---
aliases:
  - /posts/tiny-networking-in-swift.html
date: 2014-11-07
title: A tiny networking library
headline: Composing functions out of functions
---


For a new project we've been working on at Unsigned Integer, the company behind [Deckset](http://www.decksetapp.com), we needed to wrap a webservice API. I had a look at both [Alamofire](https://github.com/Alamofire/Alamofire) and [Moya](https://github.com/AshFurrow/Moya). Alamofire is a full-featured networking library, and if I would need something with more features, I would almost certainly use it. Moya is also very nice, and is built on top of Alomofire, and provides some extra safety. However, neither of them provide a simple way to parse results, so I started to experiment a bit. The library's source is [available as a gist](https://gist.github.com/chriseidhof/26bda788f13b3e8a279c).

My design goal was to have a way of describing API endpoints in such a way that I don't have to use any strings when calling them, and I wanted both the input data and output data converted automatically. After a couple of iterations, I ended up with a simple struct to describe a resource that returns a value of type `A`. It contains an endpoint's path, the HTTP method, an optional request body, and some headers specific to the request. The last parameter is the most interesting: it's a way to parse the response: a function that produces an `A` from the response data. Because this parsing might fail, the result is an optional.

    struct Resource<A> {
        let path: String
        let method : Method
        let requestBody: NSData?
        let headers : [String:String]
        let parse: NSData -> A?
    }

As an example, we can construct a resource for the Github [zen endpoint](https://developer.github.com/guides/getting-started/). It says the path is "zen", it uses the `GET` method, has no request body or headers, and simply parses the data as an UTF-8 encoded string:

    func parseString(data: NSData) -> String? {
        return NSString(data: data, encoding: NSUTF8StringEncoding)
    }

    func zen() -> Resource<String> {
        return Resource(path: "zen", method: Method.GET, 
                        requestBody: nil, headers: [:], parseString)
    }

To fetch this resource, there's another function called `apiRequest`. As we will see later on, we will wrap this function for our purposes, but for now, we'll call it directly. The implementation is not too important, in order to understand it we only need to look at the type signature. It takes a `modifyRequest` function, which we'll ignore, a base URL (e.g. `https://api.github.com`), a resource (e.g. `zen()`), and then it takes two callback functions: one for the failure case, which provides a reason and an optional response body, and one for the success case. The completion handler is only called when the request returns a status code of 200, and when the parse function returns a non-nil result.

    func apiRequest<A>(modifyRequest: NSMutableURLRequest -> (), 
                       baseURL: NSURL,
                       resource: Resource<A>,
                       failure: (Reason, NSData?) -> (),
                       completion: A -> ())

Now we can construct a call. At first, we don't need to modify the request, so we just pass in a function that does nothing. For the failure handler, we define a function that logs the failure to the console.

    func defaultFailureHandler(failureReason: Reason, data: NSData?) {
        let string = NSString(data: data!, encoding: NSUTF8StringEncoding)
        println("Failure: \(failureReason) \(string)")
    }

    let baseURL = NSURL(string: "https://api.github.com")!

    apiRequest({ _ in }, baseURL, zen(), defaultFailureHandler) { message in
        println("Got a message: \(message)")
    }

If we run this code, we'll get the following output:

    Got a message: Encourage flow.
    
This works perfectly. Now, let's try our hand at something a bit more difficult: getting and parsing a JSON result. In this case, we set some headers (the content-type, and the authorization token). Then instead of passing in the `parseString` function that we used earlier, we pass in the `decodeJSON` function.

    func decodeJSON(data: NSData) -> JSONDictionary? {
        return NSJSONSerialization.JSONObjectWithData(data, options: 
                  NSJSONReadingOptions.allZeros, error: nil) 
          as? [String:AnyObject]
    }

    func authenticatedUser() -> Resource<JSONDictionary> {
        let headers = ["Content-Type": "application/json"]
        return Resource(path: "user", method: Method.GET, requestBody: nil, 
                        headers: headers, decodeJSON)
    }
    
Because we expect to have lots of times where we would like to encode and decode JSON for an API call, we can make a *smart constructor*. This is a wrapper around the `Resource` init function which will do the extra work of encoding JSON, decoding JSON and setting the right headers:

    public func jsonResource<A>(path: String, method: Method, requestParameters: JSONDictionary, parse: JSONDictionary -> A?) -> Resource<A> {
        let f = { decodeJSON($0) >>= parse }
        let jsonBody = encodeJSON(requestParameters)
        let headers = ["Content-Type": "application/json"]
        return Resource(path: path, method: method, requestBody: jsonBody, headers: headers, parse: f)
    }

Now, we can rewrite our `authenticatedUser` as follows:

    func authenticatedUser() -> Resource<JSONDictionary> {
        return jsonResource(path: "user", method: Method.GET, 
                            requestParameters: [:]) { $0 }
    }

However, instead of a dictionary, we would really like to parse this into a `GithubProfile` value:

    struct GithubProfile {
        let login: String
        let id: Int
        let avatarURL: NSURL
    }
    
    let makeGithubProfile = { GithubProfile(login: $0, id: $1, avatarURL: $2) }

We can use the [JSON parsing](/posts/json-parsing-in-swift.html) technique to write a small parsing function and rewrite our `authenticatedUser` function:

    func parseGithubProfile(dict: JSONDictionary) -> GithubProfile? {
        return curry(makeGithubProfile) <*> string(dict, "login")
                                        <*> int(dict, "id")
                                        <*> url(dict, "avatar_url")
    }
    
    func authenticatedUser() -> Resource<GithubProfile> {
        return jsonResource("user", .GET, [:], parseGithubProfile)
    }

For me, this is the power of functional programming. We wrote one `apiRequest` function and one `Resource` datatype, and never changed it again. We can provide this in a library. We then created a couple of small functions for encoding and decoding JSON, and wrapped the existing functions to create our own convenience functions. The `authenticatedUser` is now very tiny.

As a final step, we'll wrap the `apiRequest` function in a new function that's tied to the Github API. It sets the authorization token, and provides the base URL. The completion handler is called only in case of success:

    func request<A>(resource: Resource<A>, completionHandler: A -> ()) {
        func setAuthToken(request: NSMutableURLRequest) {
            request.setValue("token \(authorizationToken)", forHTTPHeaderField: "Authorization")
        }
        let baseURL = NSURL(string: "https://api.github.com")!
        apiRequest(setAuthToken, baseURL, resource, defaultFailureHandler, completionHandler)
    }

Now we can test our API with the following simple statement:

    request(authenticatedUser()) { user in
        println("User's avatar URL: \(user.avatarURL)")
    }

I think designing a library like this has a lot of benefits. We provided some base functionality, and then by composing and wrapping functions into new functions we quickly made it very specific to our app without having to put in much effort. There are no complicated configuration steps, just simple function composition.

If you want to learn more about how to design APIs in a functional way, do consider reading our book on [Functional Programming in Swift](http://www.objc.io/books/).



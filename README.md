Hydra is full-featured lightweight library which allows you to write better async code in Swift 3+. It's partially based on JavaScript A+ specs and also implements modern construct like `await`.

With Hydra your code will be cleaner, easy to mantain and sexy as ever.

## What's a Promise?
A Promise is a way to represent a value that will exists, or will fail with an error, at some point in the future. You can think about it as a Swift's `Optional`: it may or may not be a value. A more detailed article which explain how Hydra was implemented can be found here.

Each Promise is strong-typed: this mean you create it with the value's type you are expecting for and you will be sure to receive it when Promise will be resolved (the exact term is `fulfilled`).

A Promise is, in fact, a proxy object; due to the fact the system knows what success value look like, composing asynchronous operation is a trivial task; with Hydra you can:

- create a chain of dependent async operation with a single completion task and a single error handler.
- resolve many independent async operations simultaneously and get all values at the end
- retry or recover failed async operations
- write async code as you may write standard sync code
- resolve dependent async operations by passing the result of each value to the next operation, then get the final result
- avoid callbacks, pyramid of dooms and make your code cleaner!

## Create a Promise
Creating a Promise is trivial; you need to specify the `context` (a GCD Queue) in which your async operations will be executed in and add your own async code as `body` of the Promise.

This is a simple async image downloader:

```swift
func getImage(url: String) -> Promise<UIImage> {
	return Promise<UIImage>(.background, { resolve, reject in
		self.dataTask(with: request, completionHandler: { data, response, error in
        if let error = error {
            reject(error)
        } else if let data = data, let response = response as? HTTPURLResponse {
            fulfill((data, response))
        } else {
            reject("Image cannot be decoded")
        }
    }).resume()
	}
}
```

You need to remember only few things:

- a Promise is created with a type: this is the object's type you are expecting from it once fulfilled. In our case we are expecting an `UIImage` so our Promise is `Promise<UIImage>` (if a promise fail returned error must be conform to Swift's `Error` protocol)
- your async code (defined into the Promise's `body`) must alert the promise about its completion; if you have the fulfill value you will call `resolve(yourValue)`; if an error has occurred you can call `reject(occurredError)` or throw it using Swift's `throw occurredError`.
- the `context` of a Promise define the Grand Central Dispatch's queue in which the async code will be executed in; you can use one of the defined queues (`.background`,`.userInitiated` etc. [Here you can found](http://www.appcoda.com/grand-central-dispatch/) a nice tutorial about this topic)

## How to use a Promise
Using a Promise is even easier.  
You can get the result of a promise by using `then` function; it will be called automatically when your Promise fullfill with expected value.
So:

```swift
getImage(url).then(.main, { image in
	myImageView.image = image
})
```

As you can see even `then` may specify a context (by default - if not specified - is the main thread): this represent the GCD queue in which the code of the then's block will be executed (in our case we want to update an UI control so we will need to execute it in `.main` thread).

But what happened if your Promise fail due to a network error or if the image is not decodable? `catch` func allows you to handle Promise's errors (with multiple promises you may also have a single errors entry point and reduce the complexity).

```swift
getImage(url).then(.main, { image in
	myImageView.image = image
}).catch(.main, { error in
	print("Something bad occurred: \(error)")
})
```

## Chaining Multiple Promises
Chaining Promises is the next step thought mastering Hydra. Suppose you have defined some Promises:

```swift
func loginUser(_ name:String, _ pwd: String)->Promise<User>
func getFollowers(user: User)->Promise<[Follower]>
func unfollow(followers: [Follower])->Promise<Int>
```

Each promise need to use the fulfilled value of the previous; plus an error in one of these should interrupt the entire chain.  
Doing it with Hydra is pretty straightforward:

```swift
loginUser(username,pass).then(getFollowers).then(unfollow).then { count in
	print("Unfollowed \(count) users")
}.catch { err in
	// Something bad occurred during these calls
}
``` 

Easy uh? (Please note: in this example context is not specified so the default `.main` is used instead).

## Await
Have you ever dream to write asynchronous code like its synchronous counterpart? Hydra was heavily inspired by [Async/Await specification in ES8 (ECMAScript 2017) ](https://github.com/tc39/ecmascript-asyncawait)which provides a powerful way to write async doe in a sequential manner.

Using `await` with Hydraw's Promises is pretty simple: for example the code above can be rewritten directly as:

```swift
do {
	let loggedUser = try await(loginUser(username,pass))
	let followersList = try await(getFollowers(loggedUser))
	let countUnfollowed = try await(unfollow(followersList))
	print("Unfollowed \(count) users")
} catch {
	print("Something bad has occurred \(error)")
}
```

Like magic! Your code will run in `.background` thread and you will get the result of each call only when it will be fulfilled. Async code in sync sauce!
(You can however pick your custom GCD queue).

You can also use `await` with your own block:

```swift
print("And now some intensive task...")
let result = try! await(.background, { resolve,reject in
	delay(10, context: .background, closure: { // jut a trick for our example
		resolve(5)
	})
})
print("The result is \(result)")
```

## Advanced Usage
Because promises formalize how success and failure blocks look, it's possible to build behaviors on top of them.
Hydra supports:

- `always`: allows you to specify a block which will be always executed both for `fulfill` and `reject` of the Promise
- `ensure`: allows you to specify a predica block; if predicate return `false` the Promise fails.
- `timeout`: add a timeout timer to the Promise; if it does not fulfill or reject after given interval it will be marked as rejected.
- `all`: create a Promise that resolved when the list of passed Promises resolves (promises are resolved in parallel). Promise also reject as soon as a promise reject for any reason.
- `any`: create a Promise that resolves as soon as one passed from list resolves. It also reject as soon as a promise reject for any reason.
- `delay`: delay the execution of a Promise by a given time interval.
- `forward`: Perform an operation in the middle of a chain that does not affect the resolved value but may reject the chain.
- `recover`: Allows recovery of a Promise by returning another Promise if it fails.
- `zip`: Create a Promise tuple of a two promises

### always
`always` func is very useful if you want to execute code when the promise fulfills — regardless of whether it succeeds or fails.

```swift
showLoadingHUD("Logging in...")
loginUser(username,pass).then { user in
 print("Welcome \(user.username)")
}.catch { err in
 print("Cannot login \(err)")
}.always {
 hideLoadingHUD()
}
```

### ensure
`ensure` is a func that takes a predicate, and rejects the promise chain if that predicate fails.

```swift
getAllUsersResponse().ensure { httpResponse in
	guard let httpResponse.statusCode == 200 else {
		return false
	}
	return true
}.then { usersList in
	// do something
}.catch { error in
	// request failed, or the status code was != 200
}

```

### timeout
`timeout` allows you to attach a timeout timer to a Promise; if it does not resolve before elapsed interval it will be rejected with `.timeoutError`.

```swift
loginUser(username,pass).timeout(.main, 10, .MyCustomTimeoutError).then { user in
	// logged in
}.catch { err in
	// an error has occurred, may be `MyCustomTimeoutError
}
```

### all
`Promise.all` is a static method that waits for all the promises you give it to fulfill, and once they have, it fulfills itself with the array of all fulfilled values (in order).

If one Promise fail the chain fail with the same error.

Execution of all promises is done in parallel.

```swift
let promises = usernameList.map { return getAvatar(username: $0) }
Promise.all(promises).then { usersAvatars in
	// you will get an array of UIImage with the avatars of input
	// usernames, all in the same order of the input.
	// Download of the avatar is done in parallel in background!
}.catch { err in
	// something bad has occurred
}
```

### any
`any` easily handle race conditions: as soon as one Promise of the input list resolves the handler is called and will never be called again.

```swift
let mirror_1 = "https://mirror1.mycompany.com/file"
let mirror_2 = "https://mirror2.mycompany.com/file"

any(getFile(mirror_1), getFile(mirror_2)).then { data in
	// the first fulfilled promise also resolve the any Promise
	// handler is called exactly one time!
}
```


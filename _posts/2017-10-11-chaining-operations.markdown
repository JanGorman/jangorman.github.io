---
layout: post
title: "Chaining Operations"
date: 2017-10-11T10:15:53+02:00
categories: [Swift, Operation, OperationQueue, NSOperation, NSOperationQueue]
---

Today it's time to take a closer look at [Operation](https://developer.apple.com/documentation/foundation/operation) and how they can be used to perform work in the background. Compared to [GCD](https://apple.github.io/swift-corelibs-libdispatch/), Operations are pretty useful for tasks that you might want to cancel or for tasks that are dependent on one another. What's maybe less obvious though is how to pass data between dependent operations. This is fairly trivial to achieve by introducing "glue"-operations whose sole purpose it is to shuffle data from one operation to the other. The system defined [BlockOperation](https://developer.apple.com/documentation/foundation/blockoperation) subclass can be used to do just that.

### A Fetch Operation

Assume that we need to fetch _something_ from an API. For illustration purposes, let's use [JSONPlaceholder](https://jsonplaceholder.typicode.com) to fetch some users. Let's define a basic user struct first:

```swift
// If you want to follow along in a Swift Playground
// import PlaygroundSupport

// PlaygroundPage.current.needsIndefiniteExecution = true

struct User: Codable {
  let id: Int
  let name: String
  let email: String
}
```

Next is our `FetchOperation` subclass that pulls in a list of users:

```swift


final class FetchOperation: Operation {

  override var isAsynchronous: Bool {
    return true
  }

  // Let our backing variables emit KVO notifications
  private var _isExecuting = false {
    willSet {
      willChangeValue(forKey: "isExecuting")
    }
    didSet {
      didChangeValue(forKey: "isExecuting")
    }
  }
  override var isExecuting: Bool {
    return _isExecuting
  }

  private var _isFinished = false {
    willSet {
      willChangeValue(forKey: "isFinished")
    }
    didSet {
      didChangeValue(forKey: "isFinished")
    }
  }
  override var isFinished: Bool {
    return _isFinished
  }

  private(set) var users: [User]?
  private(set) var error: Error?

  private let url: URL
  private var task: URLSessionTask?

  init(url: URL) {
    self.url = url
    super.init()
  }

  override func start() {
    // For asynchronous operations, check the isCancelled state before performing work
    guard !isCancelled else { return }

    task = URLSession.shared.dataTask(with: url) { [weak self] data, _, error in
      defer {
        self?._isFinished = true
      }
      self?.error = error

      guard let data = data,
            let users = try? JSONDecoder().decode([User].self, from: data) else { return }
      self?.users = users
    }
    task?.resume()
    _isExecuting = true
  }

  override func cancel() {
    task?.cancel()
    _isFinished = true
  }

}
```

That's quite a chunk of code. Let's step through it. For concurrent (read asynchronous) operations, you at least need to override `start()`, `isAsynchronous`, `isExecuting` and `isFinished`. `isAsynchronous` is easy, just return `true`. The other variables are a bit more tricky since they should also send out KVO notifications. The way I do it here is by adding a private backing variable in each case and using their respective `willSet` and `didSet` handlers to emit the notifications for the correct keyPath.

The `start()` method itself then is quite straightforward. A quick sanity check that the operation wasn't cancelled and then it uses `URLSession` to asynchronously call our API.

Extra points for implementing the `cancel()` method. In our case, cancel the optional `URLSessionTask` and also remember to set the `isFinished` status. Even if you don't perform the work you still need to correctly set this property.

### The Process Operation

The dumbest example I could come up with is for the `ProcessOperation` to print out the users fetched before. This should look very familiar now:

```swift
final class ProcessOperation: Operation {

  override var isAsynchronous: Bool {
    return true
  }

  private var _isExecuting = false {
    willSet {
      willChangeValue(forKey: "isExecuting")
    }
    didSet {
      didChangeValue(forKey: "isExecuting")
    }
  }
  override var isExecuting: Bool {
    return _isExecuting
  }

  private var _isFinished = false {
    willSet {
      willChangeValue(forKey: "isFinished")
    }
    didSet {
      didChangeValue(forKey: "isFinished")
    }
  }
  override var isFinished: Bool {
    return _isFinished
  }

  var users: [User]?
  var error: Error?

  override func start() {
    guard !isFinished, let users = users, error == nil else { return }

    for user in users {
      print(user)
    }
  }

  override func cancel() {
    _isFinished = true
  }

}
```

### Glue

To glue the two together now:

```swift
// Our first operation that fetches something from an API
let url = URL(string: "https://jsonplaceholder.typicode.com/users")!
let fetch = FetchOperation(url: url)
// Our second operation that needs the result of the first API call
let process = ProcessOperation()
// And finally our glue operation to tie the two together
let glue = BlockOperation {
  process.users = fetch.users
  process.error = fetch.error
}
// Make process depend on glue
process.addDependency(glue)
// And Glue depend on fetch, thereby maintaing the right order
glue.addDependency(fetch)

// Add all operations to an operation queue
let queue = OperationQueue()
queue.addOperations([fetch, process, glue], waitUntilFinished: false)
```

To summarise, both the `fetch` and `process` are custom subclasses of `Operation` that do some units of work. For the `glue` we just use a `BlockOperation`, anything more would be overkill. This design is very flexible since it allows us to re-use operations and compose them for specific tasks in any order we need.

Adding more operations into the mix is as easy as creating further Operations, hooking up the dependencies between them and adding them to our operation queue.

What's also really cool about Operations and [OperationQueue](https://developer.apple.com/documentation/foundation/operationqueue) is that you can use multiple queues and yet still have individual operations depend on one another.

Hope you found this useful, feel free to reach out to me via [Twitter](https://twitter.com/JanGorman).
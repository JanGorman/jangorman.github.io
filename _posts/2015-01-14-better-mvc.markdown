---
layout: post
title: "Better MVC"
date: 2015-01-14 21:25:14 +0100
comments: true
categories: [Swift, iOS, Design Patterns]
---

I had been meaning to write this for a while now. And even if the examples are all in Swift (because shiny) the same can be applied to Objective-C code as well without any problem.

Slimming down your iOS view controllers is a big topic around the office and there was even an entire issue of [objc.io](http://www.objc.io/issue-1/) centered around the topic (as well as another one on [architecture](http://www.objc.io/issue-13/)).

We had also tried some [MVVM](http://www.objc.io/issue-13/mvvm.html) as well as [Viper](http://www.objc.io/issue-13/viper.html) (ok, being an avid Uncle Bob [clean code](https://cleancoders.com) watcher and reader, I know this is about a lot more but imho doesn't apply too well to iOS apps since there is only *one* presenter) but neither of those models really stuck. If you look at what Wikipedia has to say about [MVC](http://en.wikipedia.org/wiki/Model–view–controller) you will notice that it mentions the model notifying its associated views and controllers. Aha! So why not try that:

```swift
class SimpleView: UIView {
  …
  @IBOutlet weak var nameTextField: UITextField!
  weak var model: SimpleModel! {
      didSet {
          model.addObserver(self, forKeyPath: "name", options: .New, context: &context)
      }
  }
  
  …
  
  override func observeValueForKeyPath(keyPath: String, ofObject object: AnyObject, change: [NSObject:AnyObject],
                                       context: UnsafeMutablePointer<Void>) {
      if context == &self.context {
          nameLabel.text = "Hi there, \(change[NSKeyValueChangeNewKey]!)"
      } else {
          super.observeValueForKeyPath(keyPath, ofObject: object, change: change, context: context)
      }
  }
  
  @IBAction func didSubmit(sender: AnyObject) {
      controller.didSubmitName(nameTextField.text)
  }
}
```

The view takes care of all it's own presentation and not the controller (as you'll see in code most of the time). Now of course KVO can get pretty cumbersome when there are a lot of properties involved but you get the point. The controller has nothing to do with the presentation of the model and just received requests from the view to update the model. What does the controller look like?

```swift
protocol SimpleController: class {

    func didSubmitName(name: String)

}

class SimpleViewController: UIViewController, SimpleController {

    let model = SimpleModel()

    override func viewDidLoad() {
        super.viewDidLoad()

        let mainView = self.view as SimpleView
        mainView.controller = self
        mainView.model = model
    }

    func didSubmitName(name: String) {
        model.name = name
    }

}
```

Ok, very simple example indeed. What about something more involved:

```swift
class AdvancedView: UIView, UserModelObserver, RemoteModelObserver {
    …
    weak var userModel: UserModel! {
        didSet {
            userModel.addObserver(self)
        }
    }
    weak var remoteModel: RemoteModel! {
        didSet {
            remoteModel.addObserver(self)
        }
    }
    …  
}
```

As you can see, the model implements two protocols this time and listens to two different models for changes. And again, the controller is very slim:

```swift
class AdvancedViewController: UIViewController {

    let userModel = UserModel()
    let remoteModel = RemoteModel()

    override func viewDidLoad() {
        super.viewDidLoad()

        let advancedView = view as AdvancedView
        advancedView.controller = self
        advancedView.userModel = userModel
        advancedView.remoteModel = remoteModel
    }

    func submit(#firstName: String, lastName: String) {
        userModel.firstName = firstName
        userModel.lastName = lastName
        userModel.validate()
    }

    func load() {
        remoteModel.load()
    }

}
```

Any validation logic is done inside the models (which makes it easily testable). Code for loading remote resources is also done in the model:

```swift
func load() {
    if let URL = NSURL(string: Constants.URLString) {
        let session = NSURLSession.sharedSession()
        let task = session.dataTaskWithURL(URL) {
            (data, response, error) in
            if error != nil {
                println("Epic Fail")
                return
            }

            var jsonError: NSError?
            let json = NSJSONSerialization.JSONObjectWithData(data, options: NSJSONReadingOptions.AllowFragments,
                    error: &jsonError) as NSArray
            self.comments = json
            dispatch_async(dispatch_get_main_queue()) {
                self.notifyObservers()
            }
        }
        task.resume()
    }
}
```

I've uploaded a sample project over on [github](https://github.com/JanGorman/BetterMVC) including a third example of a `UITableViewController` and some test code.
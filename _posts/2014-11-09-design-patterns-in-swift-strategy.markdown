---
layout: post
title: "Design Patterns in Swift: Strategy"
date: 2014-11-09 10:44:39 +0100
comments: true
categories: [Swift, iOS, Design Patterns]
---

As a refresher, I thought it could be fun to look at some common design patterns and how they apply to Swift. One very essential part of the pattern-tool-chest is the strategy pattern. Let's dive straight in.

With the strategy pattern you define algorithms or behavior. You then compose your objects out of those behaviors. How does that look like?

First, your behavior protocol and two possible implementations:

```swift
protocol TapBehavior {
    
    func tap()
    
}

class CartTapBehavior: TapBehavior {
    
    func tap() {
        println("I'm the cart tap")
    }
    
}

class WishListTapBehavior: TapBehavior {
    
    func tap() {
        println("I'm the wish list tap")
    }
    
}
```

Which brings up an important point: it's always preferrable to program to a protocol/interface rather than to a concrete implementation.

Next, you'd want to actually use the behavior:

```swift
class SomeViewWithAButton: UIView {
    
    @IBOutlet weak var aButton: UIButton!
    
    var tapBehavior: TapBehavior?
    
    @IBAction func didTapButton(sender: AnyObject) {
        tapBehavior?.tap()
    }
    
}
```

By making the behavior a property of the class it is now easy to add and replace behaviors at runtime. You might begin to see how this can lead to very flexible designs. A different approach here could have been to create an abstract base view and then create two concrete implementations for different tap behaviors but as soon as you need a new kind of behavior you'd be forced to create a new subclass. And now imagine that you have  multiple behaviors in a class – the subclass approach would lead to all kinds of combinations of subclasses that each have to implement variations of behavior.

Even if this is an extremely simple pattern, it's good to keep it in mind the next time you have to implement varying behavior in some kind of base class. Would composing work better than inheriting?

A full example is available as Xcode playground on [github](https://github.com/JanGorman/Swift-Design-Patterns)

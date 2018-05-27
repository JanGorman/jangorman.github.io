---
layout: post
title: "Design Patterns in Swift: Adapter Pattern"
date: 2014-12-01 20:44:17 +0100
comments: true
categories: [Swift, iOS, Design Patterns]
---

Already December! Time for a new chapter on applying design patterns in Swift. This time it's the [Adapter Pattern's](http://en.wikipedia.org/wiki/Adapter_pattern) turn. Good stuff! And simple!

So what is it and what can it do for you? The Adapter Pattern converts the interface of one class into another interface. That's it. And with this simple pattern you can let classes work together that otherwise couldn't.

So let's assume you have the following:

```swift
protocol MusicPlayer {

    func insertMedia()
    func play()

}

class CassettePlayer: MusicPlayer {

    func insertMedia() {
        println("Insert mixtape")
    }

    func play() {
        println("Push down clunky button and play")
    }

}
```

Clearly, this kind if thing is only [acceptable in the 80s](https://www.youtube.com/watch?v=dIm87r9lnD0).

Fast forward to now, Spotify, beats, you name it. How do you travel back in time and get a client that accepts a `MusicPlayer` to stream and play a track? You adapt it:

```swift
protocol MediaPlayer {

    func getMedia(title: String)
    func play()

}

class StreamingMediaPlayer: MediaPlayer {

    func getMedia(title: String) {
        println("Acquiring \(title) from the cloud")
    }

    func play() {
        println("Playing!")
    }

}

// MARK: Adapter in action

class StreamingMediaPlayerAdapter: MusicPlayer {

    let player: StreamingMediaPlayer

    init(player: StreamingMediaPlayer) {
        self.player = player
    }

    func insertMedia(title: String) {
        player.getMedia(title)
    }

    func play() {
        player.play()
    }

}

```

Jaws drop in amazement. But now consider your existing project that you want to spice up with some of that Swift. You've written your amazing new classes, time to integrate with Objective-C. So you head over to [Using Swift with Cocoa and Objective-C](https://developer.apple.com/library/ios/documentation/swift/conceptual/BuildingCocoaApps/MixandMatch.html#//apple_ref/doc/uid/TP40014216-CH10-XID_77) only to find that the new goodness doesn't all work. All of these aren't compatible with Objective-C:

- Generics
- Tuples
- Enumerations defined in Swift
- Structures defined in Swift
- Top-level functions defined in Swift
- Global variables defined in Swift
- Typealiases defined in Swift
- Swift-style variadics
- Nested types
- Curried functions

Not to despair though, you can still go ahead and be as idiomatic in Swift as you want. With some extra effort you can get the old to talk to the new. Consider a Swift only protocol that just needs to use a tuple:

```swift
protocol Legend {

    func doSomething() -> (String, String)

}

class IAmLegend: Legend {

    func doSomething() -> (String, String) {
        return ("very", "important")
    }

}

func doSomethingWithLegend(legend: Legend) {
    let (first, second) = legend.doSomething()
    // Do something else with these variables
}
```

Important stuff going on. You get the idea. But obviously you won't be able to call that from any existing code.


```swift
@objc class Compatible {

    let first: String
    let second: String

    init(first: String, second: String) {
        self.first = first
        self.second = second
    }

}

class LegendSwiftAdapter: Legend {

    let compatible: Compatible
    
    init(compatible: Compatible) {
        self.compatible = compatible
    }
    
    func doSomething() -> (String, String) {
         return (compatible.first, compatible.second)
    }

}

@objc class LegendObjcAdapter {

    func doSomethingWithATuple(compatible: Compatible) {
        let adapter = LegendSwiftAdapter(compatible)
        doSomethingWithLegend(adapter)
    }

}

```

I hope it's somewhat clear what's going on. Contrived example maybe… I'll upload a more practical application soon-ish. Anyways, that's the Adapter Pattern applied in Swift for you. As always the code is available as Xcode playground on [github](https://github.com/JanGorman/Swift-Design-Patterns).
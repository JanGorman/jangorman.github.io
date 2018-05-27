---
layout: post
title: "Design Patterns in Swift: Command"
date: 2014-11-17 21:19:18 +0100
comments: true
categories: [Swift, iOS, Design Patterns]
---

Continuing on in the series, lets have a look at the Command Pattern and how to implement it in Swift. As [Wikipedia](http://en.wikipedia.org/wiki/Command_pattern) so eloquently defines it

> In object-oriented programming, the command pattern is a behavioral design pattern in which an object is used to represent and encapsulate all the information needed to call a method at a later time. 

The encapsulation part is key. Say you have a bunch of different objects that can all do _something_ and you want to expose those in a unified kind of way then the command pattern is your friend. Another great feature is that it allows for undos of those actions as you can keep a stack of the commands you ran and just pop them off the stack with an undo method. So what does that look like? First you define your command protocol:

```swift
protocol Command {

    func execute()
    func undo()

}
```

Easy enough. Lets get concrete:

```swift
class Light {
    
    func on() {
        println("The light is on")
    }
    
    func off() {
        println("The light is off")
    }
    
}

class LightCommand: Command {
    
    let light: Light
    
    init(light: Light) {
        self.light = light
    }
    
    func execute() {
        light.on()
    }
    
    func undo() {
        light.off()
    }
    
}
```

Apart from producing a bunch of code that wraps the obvious nothing magic has happened yet. But now imagine you have some other appliance that also has an on/off kind of functionality but with different method signatures:

```swift
class Heating {
    
    func turnUp(degrees: Int) {
        println("The heating is set to \(degrees)°C")
    }
    
    func turnOff() {
        println("The heating is off")
    }

}

class HeatingCommand: Command {
    
    let heating: Heating
    
    init(heating: Heating) {
        self.heating = heating
    }
    
    func execute() {
        heating.turnUp(23)
    }
    
    func undo() {
        heating.turnOff()
    }
    
}
```

Metric of course.

And it gets neater: There's of course nothing to stop you from bunching together multiple method calls into a single command. So to continue with the home automation theme, you could have a macro type command `ArriveAtHomeCommand` that switches on a bunch of lights, sets the heating to a comfortable level and switches on the TV.

Too much code! More  cryptic! Ok, ok. So you can also  do a similar thing using closures that are  your commands. The simplified
way to define a closure makes that at least a bit more readable than it was in Objective-C:

```swift
typealias CommandClosure = () -> Void

var commands = [CommandClosure]()

let heatingCommand = { () -> Void in
    let heating = Heating()
    heating.turnUp(23)
}

commands.append(heatingCommand)

// Much later
commands[0]()
```

That also works. The option to store closures/functions inside of arrays is pretty cool but here you loose the build in undo option. The way around that would be to define both an execute and undo block and whenever you execute a command you push its corresponding undo into your array/stack. But if you're only going one way and can skip the undo, this is a viable way to go.

Again the code is available as Xcode playground on [github](https://github.com/JanGorman/Swift-Design-Patterns).
---
layout: post
title: "iOS Storage"
date: 2014-08-31 11:57:36 +0200
comments: false
categories: [Objective-C, iOS, Persistence]
---

When it comes to persistence for iOS there are a few options to choose from:

* [Core Data](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CoreData/cdProgrammingGuide.html)
* [SQLite](http://www.sqlite.org)
* [NSKeyedArchiver](https://developer.apple.com/library/mac/documentation/cocoa/reference/Foundation/Classes/NSKeyedArchiver_Class/Reference/Reference.html)
* [FastCoding](https://github.com/nicklockwood/FastCoding)
* [YapDatabase](https://github.com/yaptv/YapDatabase)
* [Realm](http://realm.io)

Which one you end up using of course depends on your use case and each of the solutions has some tradeoff with regards to read/write performance and implementation complexity.

### Background

At work we had been using Core Data with [Magical Record](https://github.com/magicalpanda/MagicalRecord) shielding us from some of the gory implementation details when it comes to wrangling with different `NSManagedObjectContext` and getting out of the users main thread.

Another promise of MagicalRecord are "free" lightweight core data migrations using `[MagicalRecord setupCoreDataStackWithAutoMigratingSqliteStoreNamed:@"storage.sqlite"]` and so one fateful release we decided to kill off an old, unused model. Prior tests revealed no issues going ahead with the migration but a certain combination of being logged in to the app and adding something to your shopping cart resulted in the dreaded `Can't merge models with two different entities named …` error. A revert and a fast-track approval by Apple saved the day but confidence in MR and Core Data was… a bit shaken.


### Enter Yap

I had privately been playing around with [YapDatabase](https://github.com/yaptv/YapDatabase) and liked it. A lot. Concurrency, views, it being a key-value storage built on top of SQLite and especially the concept of `YapDatabaseViewRowChange` for updating table views made for compelling features.

So I pitched the idea at work and we migrated over some Core Data tables to Yap. One thing that became obvious during the development was that Yap requires quite a bit of boilerplate code to set up. You always end up creating views, long lived transactions and managing `NSNotification` to update said transactions. But I rationalized it as being a one time effort and chugged along.

After the release our Crashlytics popped up with an unnerving number of Yap related crashes. Not all, but most having to do with notifications being sent to deallocated observers. This seemed rather strange as we were removing the observer correctly on deallocation and never found a root cause. Stability of the app got so bad though that a hotfix release was scheduled and Yap was replaced.

### Enter NSKeyedArchiver

We went back to the drawing board and considered our options. We didn't want to deal with Core Data's complexity any longer and reliance on complex third party libraries had left a sour taste. What do we know:

* Storage has to run in the background and not block the main thread
* We don't have any big tables… the biggest one needs to store 100 entries. (Others are out of our control because customer's fill them so a huge buffer would set an upper limit of 500 entries)
* It needs to be simple, every one on the team should immediately know what's going on
* No third party library if possible
* No need to store anything in iCloud, any syncing happens between the app and our own servers

`NSKeyedArchiver` gets a bad wrap for being slow, and yes, try to store a lot of values (in the thousands!) and it degrades pretty badly but for the use case stated above of maybe 500 entries it actually is pretty decent. Xcode 6 adds the handy `[self measureBlock:^{}]`, time for some benchmarks.

* Yap scored a write performance of 0.005 seconds. Great result
* FastCoding came in a respectable second with 0.013 seconds
* NSKeyedArchiver scored last with 0.110 seconds

But, and this is a huge but, it is fast enough for our needs. The database (or essentially just an `NSArray`) is read on into memory and any changes to the data also happen in memory and it actually is very snappy to use. Persisting to disk is done on a background thread using an `NSOperationQueue` with a size of one so all additions and deletes queue up nicely and there are no nasty concurrency issues to deal with.

A neat side effect was that we also don't need separate models for Core Data and our API anymore. We use [Mantle](https://github.com/Mantle/Mantle) extensively and the cool thing is that all `MTLModel` subclasses also comply to `<NSCoding>`. We can pass the same values back and forth.

### Learning

* Always know your real use case
* Benchmark!
* Don't rely on third party libraries as much. Cocoapods are _awesome_ but sometimes simple does it.
* Find edge cases during your QA phase
---
layout: post
title: "Automated Canary Builds"
date: 2015-03-15 10:59:18 +0100
comments: true
categories: [iOS, Jenkins, Build Automation, Ruby]
---

Like many iOS projects, yours probably also has external code dependencies. There's a number of options how to manage them. You can do it manually by including other projects into your workspace or automate the whole process with tools such as [Cocoapods](http://cocoapods.org), the newer [Carthage](https://github.com/Carthage/Carthage) or the little known [peru](https://github.com/buildinspace/peru), which functionally sits somewhere between manually managing dependencies and Carthage.

Whichever way you go, you should always integrate against known versions of these libraries to ensure that builds are reproducible. You'll probably want to stay up to date with newer releases to receive security and bug fixes or performance improvements but at the same time not spend too much effort on manually updating them and verifying that there aren't any bugs. This is were a nightly **Canary Build** comes into play that automatically checks for any updated libraries, creates a build, and runs your tests. A successful build indicates that it's safe for you to update.

I'll focus on Cocoapods here, since that's the one we use at work and also has a nice option to check for new versions of pods. In its simplest form a `Podfile` looks like this:

``` ruby
source 'https://github.com/CocoaPods/Specs'
pod 'AFNetworking'
```

No version information whatsoever. But running `pod install` will also generate a `Podfile.lock` (which you should *always* check in to your repository). In the past I've shied away from doing this or the loosely versioned way of defining a dependency with e.g. `pod 'AFNetworking', '~> 2'` because I wanted to be able to see which version of a library is included at a glance.

So in my case, the `Podfile` might look something like this:

``` ruby
source 'https://github.com/CocoaPods/Specs'

platform :ios, '7.0'
xcodeproj 'AwesomeProject'

pod '1PasswordExtension', '1.1.1'
pod 'AFNetworking', '2.5.0'
pod 'AFNetworkActivityLogger', '2.0.3'
pod 'CocoaLumberjack', '1.9.1'
pod 'DBCamera', '2.3.6'
pod 'TTTAttributedLabel', '1.10.2'
…
```

Cocoapods comes with the very handy `outdated` command to find out about any new versions:

``` bash
$ pod outdated

Updating spec repositories
Analyzing dependencies
The following updates are available:
- 1PasswordExtension 1.1.1 -> 1.1.2 (latest version 1.1.2)
- AFNetworkActivityLogger 2.0.3 -> 2.0.3 (latest version 2.0.4)
- AFNetworking 2.5.0 -> 2.5.0 (latest version 2.5.1)
- CocoaLumberjack 1.9.1 -> 1.9.1 (latest version 2.0.0)
- DBCamera 2.3.6 -> 2.3.6 (latest version 2.3.13)
- TTTAttributedLabel 1.10.2 -> 1.10.2 (latest version 1.13.2)
```

And then I'd usually go in by hand, update the `Podfile`, run `install` and build the project. Obviously not the best use of anyone's time. So this is where the Canary Build comes into play. As part of a [Jenkins](http://jenkins-ci.org) build script (I like ruby) you could do something like the following to automatically update a strictly versioned `Podfile`:

``` ruby
def update_pods
  updates = Hash[%x{pod outdated}
    .lines("\n")
    .select { |line| line.start_with?('-') }
    .collect { |line| 
      match = /(?<pod>[[:alnum:]]{1,}).{1,}(?:[latest version])(?<version>[0-9.]{1,})/.match(line)
      [match['pod'], match['version']]
    }
    .map { |key, value| [key, value] }
  ]

  return if updates.empty?

  lines = IO.readlines('Podfile').map do |line|
    if line.start_with?('pod')
      match = /pod\s['|"](?<pod>[[:alnum:]]{1,})["|']/.match(line)
      if !match.nil? && updates[match['pod']]
        pod = match['pod']
        line = "pod '#{pod}', '#{updates[pod]}'"
      end
    end
    line
  end

  File.open('Podfile', 'w') do |file|
    file.puts lines
  end
end
```

A little bit of regex magic to extract the pod name and new version into a hash which we can match against the `Podfile` and update if needed. Then the script would go off and run `pod update`, build and run tests and maybe even upload to HockeyApp or TestFlight and send out an email with which pods it updated.

But of course once you start automating this, there is no need to actually version the pods anymore since you'll have the generated `Podfile.lock` and/or let the script inform you about any pods it did update. In such a case it's enough to simply:

``` ruby
def update_pods
  updates = %x{pod outdated}
    .lines("\n")
    .select { |line| line.start_with?('-') }
  return if updates.empty?
  %x{pod update}
  %x{xctool test}
  # Send out list of updated pods via email
end
```

A similar approach should also be possible using Carthage if you're willing to go without fixed versions since it will generate a `Cartfile.resolved` with a fixed version used for the build.

### Wrap-up

We have several Jenkins jobs running already for different branches and pre-release versions. Adding a nightly Canary Build is a logical next step and I'd urge anyone to give it a try. The effort to set it up is fairly low but it does free up a lot of time in the long run and should also help with the overall stability of your apps. I'd love to hear about your ideas or experiences with such a process. [Get in touch on Twitter.](https://twitter.com/JanGorman)

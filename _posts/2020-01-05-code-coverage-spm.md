---
layout: post
title: "Code Coverage in Swift Package Manager projects"
date: 2020-01-05T07:38:53+02:00
categories: [Swift, SPM, Coverage, codecov, travis]
---

I've been moving more and more of my projects to SPM (Swift Package Manager). For pure SPM projects it's not obvious
how to collect code coverage and integrate reporting with something like [codecov](https://codecov.io/). Here's a quick 
guide how to configure your project and integrate it with Travis.

Instead of just running `swift test` you instead generate an xcodeproj file and then use `xcodebuild` to run tests and 
collect coverage:

```bash
$ swift package generate-xcodeproj
$ xcodebuild test -scheme {YourProject}-Package -destination \
platform="macOS" -enableCodeCoverage YES
```

This generates the coverage data you need to then integrate with codecov. The steps inside your `travis.yml` file would
look like this then:

```yml
script:
  - swift package generate-xcodeproj
  - xcodebuild test -scheme {YourProject}-Package -destination \
  platform="macOS" -enableCodeCoverage YES
after_success:
  - bash <(curl -s https://codecov.io/bash)
```


---
layout: post
title: "Providers"
date: 2015-08-26 11:42:40 +0200
categories: [Swift]
---

As you'll know watchOS 2 allows you to create your own [complications](https://developer.apple.com/library/prerelease/watchos/documentation/General/Conceptual/AppleWatch2TransitionGuide/CreatingaComplication.html#//apple_ref/doc/uid/TP40015234-CH10-SW1). One thing that really stood out to me is the idea of using providers instead of setting either text or images directly. There's the abstract base class [CLKTextProvider](https://developer.apple.com/library/prerelease/watchos/documentation/ClockKit/Reference/CLKTextProvider_class/index.html#//apple_ref/occ/cl/CLKTextProvider) and [CLKImageProvider](https://developer.apple.com/library/prerelease/watchos/documentation/ClockKit/Reference/CLKImageProvider_class/index.html#//apple_ref/occ/cl/CLKImageProvider) that you provide (💥) to your complication. It's the context in which the complication is rendered that decides on the actual on-screen representation of information.

So there's this app I used to work on that needs to display prices a lot. And depending on the context where this prices is displayed it renders differently. Sounds similar to what the complication does. So in a grossly oversimplified example, let's build something around providers using Swift:


```swift
struct PriceTextProvider {

    let basePrice: Double
    let reducedPrice: Double
    let vat: Double

}
```

This is our base provider. For demo purposes we'll use `Double` for the prices.

Next come our options on _what_ we want to show using the new and great `OptionSetType`:

```swift
struct PriceTextRenderOptions: OptionSetType {

    let rawValue: Int

    init(rawValue: Int) {
        self.rawValue = rawValue
    }

    static let Base = PriceTextRenderOptions(rawValue: 1)
    static let Reduced = PriceTextRenderOptions(rawValue: 2)
    static let Vat = PriceTextRenderOptions(rawValue: 4)
    static let BaseAndReduced = [Base, Reduced]
    static let FullPrice = [Base, Reduced, Vat]

}
```

And context:

```swift
enum PriceTextContext {
    case Normal, Short
}
```

Of course we could also opt to just have a context enum that then actually decides which rendering options to use. Maybe we should, I'm undecided.

And now the renderer:

```swift
class PriceTextRenderer {
    
    let priceFormatter: NSNumberFormatter
    let vatFormatter: NSNumberFormatter

    init(priceFormatter: NSNumberFormatter, vatFormatter: NSNumberFormatter) {
        self.priceFormatter = priceFormatter
        self.vatFormatter = vatFormatter
    }

    func render(provider: PriceTextProvider, context: PriceTextContext, options: PriceTextRenderOptions) -> NSAttributedString {
        let attributedPrice = NSMutableAttributedString(string: "")
        let glue = context == .Normal ? "\n" : " "
        
        if let basePrice = priceFormatter.stringFromNumber(NSNumber(double: provider.basePrice)) where options.contains(.Base) {
            let a = NSAttributedString(string: basePrice, attributes: [NSForegroundColorAttributeName: UIColor.greenColor()])
            attributedPrice.appendAttributedString(a)
        }
        if let reducedPrice = priceFormatter.stringFromNumber(NSNumber(double: provider.reducedPrice)) where options.contains(.Reduced) {
            let a = NSAttributedString(string: "\(glue)\(reducedPrice)", attributes: [NSForegroundColorAttributeName: UIColor.redColor()])
            attributedPrice.appendAttributedString(a)
        }
        if let vat = vatFormatter.stringFromNumber(NSNumber(double: provider.vat)) where options.contains(.Vat) {
            let a = NSAttributedString(string: "\(glue)\(vat)")
            attributedPrice.appendAttributedString(a)
        }

        return attributedPrice
    }

}
```

There's a few things going on here. So let's brake it down:

First we're supplying the renderer with our [NSNumberFormatters](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSNumberFormatter_Class/). Still I see people displaying prices without using formatters. Stop it, you're doing it wrong. Even if your designers or you insist on really specific formats (though I'd always argue to rather go with the locale defaults), you can fine tune every last bit of the formatter to your liking.

Next up is the render method. You'll probably want to really fine tune this part. Ideas that come to mind are a `.Short` format that removes decimal places if appropriate, drops certain parts of the price and things like glue words that can be prepended or formatted in a specific way. E.g. `VAT: %s`. Or maybe work out the reduction in price and highlight that. People love a bargain.

So finally calling the whole thing:

```swift
let locale = NSLocale(localeIdentifier: "de_DE")

let priceFormatter = NSNumberFormatter()
priceFormatter.numberStyle = .CurrencyStyle
priceFormatter.locale = locale

let vatFormatter = NSNumberFormatter()
vatFormatter.numberStyle = .PercentStyle
vatFormatter.locale = locale

let renderer = PriceTextRenderer(priceFormatter: priceFormatter, vatFormatter: vatFormatter)

let provider = PriceTextProvider(basePrice: 20.00, reducedPrice: 15.00, vat: 0.19)
renderer.render(provider, context: .Normal, options: [.Base, .Reduced, .Vat])
renderer.render(provider, context: .Short, options: [.Base, .Reduced])
```

Get the whole thing as a playground over at this [github gist](https://gist.github.com/JanGorman/2a28acc110c8c9f38993).
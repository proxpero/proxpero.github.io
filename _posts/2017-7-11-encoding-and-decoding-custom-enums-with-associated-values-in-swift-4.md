---
title: Encoding and Decoding Enums with Associated Values in Swift 4
summary: Swift 4 adds the ability to easily archive <code>struct</code> and <code>enum</code> types by implementing the <code>Codable</code> protocol, but default support for <code>enum</code>s is limited to those with prepopulated default values (raw values). Conforming an <code>enum</code> with associated values to the <code>Codable</code> protocol involves extra work, and it is the purpose of this article to show how this could be done.
toc:
  - title: Apple's <code>Barcode</code> example
    link: /#barcode
  - title: The Intermediate Type
    link: /#the-intermediate-type
  - title: Making Barcode Codable
    link: /#codebarcode-codablecode
  - title: Usage
    link: /#usage
  - title: But Does It Scale?
    link: /#but-does-it-scale
  - title: Conclusion
    link: /#conclusion
tags: Swift Codable
---
One way to allow a custom enumeration with associated values to conform to the `Codable` protocol is to wrap it in a `struct` which *is* `Codable`. I'm going to show how to do this step-by-step, using the `Barcode` example from [the Enumeration section](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Enumerations.html#//apple_ref/doc/uid/TP40014097-CH12-ID145) of the [Swift Programming book](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/).

Some `enum`s come prepopulated with raw values, like for example this one in which each planet is assigned an `Int` value.
{% highlight swift %}
enum Planet: Int {
    case mercury = 1, venus, earth, mars, jupiter, saturn, uranus, neptune
}
{% endhighlight %}
In this case, archiving the `enum` is as easy as archiving an integer. But what about those `enum` types which have associated values?

{% include toc.html %}

## Barcode

Apple's `Barcode` implementation is simple:

{% highlight swift %}
enum Barcode {
    case upc(Int, Int, Int, Int)
    case qrCode(String)
}
{% endhighlight %}

This `enum` however contains more information than can be represented by a simple raw value like a `String` or an `Int` and so it cannot conform to `Codable` without additional instructions as to how each case is to be coded. One way to provide this information is to wrap the `enum` in an intermediate type that knows how to do just this.

By the way, these instructions are not something we ought to expose to wider world. Ideally, all that users need to know is that `Barcode` is `Codable`, so we'll keep the implementation details private.

## The Intermediate Type

I'm going to start by declaring a fileprivate `struct` called `Coding`, nested inside an extension of `Barcode` and conforming to `Codable`. This type needs to be able to represent all the information that is possible to represent in the original `Barcode` enum, and so presumably it will need a property for case, where type of each property matches the type of its case's associated value.
{% highlight swift %}
extension Barcode {
    fileprivate struct Coding: Codable {
        private var upc: (Int, Int, Int, Int)
        private var qrCode: String
    }
}
{% endhighlight %}
But this doesn't quite work. The point of the intermediate type is to create a representation of `Barcode` which is `Codable`, but the type of `upc` is `(Int, Int, Int, Int)` which is a tuple and tuples are not `Codable`.

One workaround to this is represent the tuple as an `Array<Int>` since collections of `Codable` types are `Codable` (tuples are not collections, nor should they be), but I'm going to create a new intermediate type for `(Int, Int, Int, Int)`: a `struct` with four `Int` properties to represent the `upc` value, and *then* we'll be back in business.
{% highlight swift %}
extension Barcode {
    fileprivate struct Coding: Codable {
        private struct UPCDigits: Codable {
            let numberSystem: Int
            let manufacturer: Int
            let product: Int
            let check: Int
        }
        private var upc: UPCDigits
        private var qrCode: String
    }
}
{% endhighlight %}

At this point, our intermediate `Barcode.Coding` type is officially `Codable`, since its properties are all `Codable`. But we're not quite done. We have a `Codable` representation of our original type, now we need methods to convert one into the other: a function to *encode* a `Barcode` instance into a `Barcode.Coding` instance and a function to *decode* a `Barcode.Coding` instance into a `Barcode`.

To do this requires one more tweak to our intermediate type. If there's one thing an `enum` is good at, it's this: An instance is one case *or* another, but *never* both. We ought to include this feature, insofar as we can, in our `Barcode.Coding` type, since we're trying to represent an `enum`. The natural way to do this is to have its properties be optional so that when a `Barcode` instance is a `.upc`, then the `qrCode` property of its `Barcode.Coding` `struct` can be `nil`, and vice versa.

We'll write an explicit `init` method in the body of the `struct` (not in an extension because we want to replace the memberwise initializer) and this `init` method will guarantee that exactly one of the properties can ever receive a value and all other properties will remain `nil`. This guarantee will provided by switching on an `enum`.

{% highlight swift %}
extension Barcode {
    fileprivate struct Coding: Codable {
        private struct UPCDigits: Codable {
            let numberSystem: Int
            let manufacturer: Int
            let product: Int
            let check: Int
        }
        private var upc: UPCDigits?
        private var qrCode: String?
        fileprivate init(barcode: Barcode) {
            switch barcode {
            case .upc(let numberSystem, let manufacturer, let product, let check):
                self.upc = UPCDigits(numberSystem: numberSystem, manufacturer: manufacturer, product: product, check: check)
            case .qrCode(let productCode):
                self.qrCode = productCode
            }
        }
    }
}
{% endhighlight %}

We now have the ability to convert a `Barcode` to a `Barcode.Coding` type which is `Codabe`, we now need implement the conversion in the other direction. We just need a method on `Barcode.Coding` that returns a `Barcode`. We'll make it `throw` for reasons that will become clear in a moment.
{% highlight swift %}
fileprivate func barcode() throws -> Barcode {
    /// TODO: provide logic.
}
{% endhighlight %}
How do we create a `Barcode` from a `Barcode.Coding`? We will switch on a tuple of all our properties, and use pattern matching to find the cases where one property value is `.some` and all the others `.none`, then return the `Barcode` instance for that case initialized with the corresponding value. We'll reach the `default` case in situations where either more than one property is non-`nil` or else all are `nil`, so then we'll need to throw an error.
{% highlight swift %}
fileprivate func barcode() throws -> Barcode {
    switch (upc, qrCode) {
    case (.some(let upcDigits), nil):
        return Barcode.upc(upcDigits.numberSystem, upcDigits.manufacturer, upcDigits.product, upcDigits.check)
    case (nil, .some(let productCode)):
        return Barcode.qrCode(productCode)
    default:
        throw CodingError.barcodeCodingError("Could not convert \(self) into either a upc or a qrCode")
    }
}
{% endhighlight %}

## <code>Barcode: Codable</code>

After all that work, we're finally ready to write the code that will conform our `Barcode` type to `Codable`. One, we need to implement two methods, one each for `Encodable` and `Decodable`, and two, we need to explicitly declare conformance.
{% highlight swift %}
extension Barcode: Codable {
    init(from decoder: Decoder) throws {
        // ??
    }
    func encode(to encoder: Encoder) throws {
        // ??
    }
}
{% endhighlight %}
Both of these methods are easy now that we've finished the hard work of creating a `Codable` type to represent a `Barcode`. The `init` method merely needs to try to create a `Barcode.Coding` instance from the `decoder` parameter, and then convert that instance into a `Barcode`.

On the other hand, the `encode` method needs to create a `Barcode.Coding` instance from `self`, and call the `encode` method of that instance with the `encoder` parameter.
{% highlight swift %}
extension Barcode: Codable {
    init(from decoder: Decoder) throws {
        self = try Barcode.Coding.init(from: decoder).barcode()
    }
    func encode(to encoder: Encoder) throws {
        try Barcode.Coding.init(barcode: self).encode(to: encoder)
    }
}
{% endhighlight %}

## Usage

Now we have a fully `Codable` type. Let's test it.

{% highlight swift %}
import Foundation

let encoder = JSONEncoder()
encoder.outputFormatting = .prettyPrinted
let decoder = JSONDecoder()
var productBarcode: Barcode
var data: Data
var json: String
var expectation: String
var result: Barcode

// upc
productBarcode = .upc(8, 85909, 51226, 3)
data = try encoder.encode(productBarcode)
json = String.init(data: data, encoding: .utf8)!
expectation = """
    {
      "upc" : {
        "check" : 3,
        "manufacturer" : 85909,
        "product" : 51226,
        "numberSystem" : 8
      }
    }
    """
result = try decoder.decode(Barcode.self, from: data)
assert(json == expectation)
assert(result == productBarcode)

// qrCode
productBarcode = .qrCode("ABCDEFGHIJKLMNOP")
data = try encoder.encode(productBarcode)
json = String.init(data: data, encoding: .utf8)!
expectation = """
    {
      "qrCode" : "ABCDEFGHIJKLMNOP"
    }
    """
result = try decoder.decode(Barcode.self, from: data)
assert(json == expectation)
assert(result == productBarcode)
{% endhighlight %}

Protip: Don't use pretty-printing for testing. Also, since we haven't actually implemented `Equatable` on our `Barcode`, we can't actually use the `==` operator. Rest assured however that they are in fact the same.

## But Does It Scale?

Even with only two cases, this was not a trivial amount of code. It's not difficult; in fact, it's very like boilerplate. With larger numbers of cases, writing all the tedious code required to get an enumeration with associated values to be `Codable` will cause you to ask the question: What am I getting for this work?

For one thing, it is likely that not all of your project's types are enums with associated values. Many will easily conform to `Codable` and you will want all your types to conform if you want any of them to. And you probably do want them to, since (1) it is in your long term interest to adopt fundamental Apple technologies in your Apple platform projects. And (2) I think Apple has  done a good job with this protocol and it is worthy to become widely adopted.

Perhaps future versions of Swift with more robust reflection abilities will make archiving workarounds like this one obsolete. I certainly would welcome that day. But until that version lands, it was right Apple to provide the limited implementation that it did now, rather than wait until it could do it "the right way". Real artists ship.

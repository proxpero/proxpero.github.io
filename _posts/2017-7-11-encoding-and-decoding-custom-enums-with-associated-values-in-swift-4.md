---
title: Encoding and Decoding Enums with Associated Values in Swift 4
summary: Swift 4 adds the ability to easily archive struct and enum types that implement the <code>Codable</code> protocol, but default support for enums is limited to those with prepopulated default values (raw values). Conforming an enum with associated values to the <code>Codable</code> protocol involves extra work, and it is the purpose of this article to show how this could be done.
toc:
  - title: Apple's <code>Barcode</code> example
    link: /#barcode
  - title: The Intermediate Type
    link: /#the-intermediate-type
  - title: Making Barcode Codable
    link: /#barcode-codable
  - title: Informal Testing
    link: /#informal-testing
  - title: But Does It Scale?
    link: /#but-does-it-scale
  - title: Improvements
    link: /improvements
  - title: The Keyed(En|De)codingContainer
    link: /#keyedendecodingcontainer
  - title: Conclusion
    link: /#conclusion
  - title: References
    link: /#references
tags: Swift Codable
updates:
  - date: 2017-7-16
    summary: Incorporated <a href="https://gist.github.com/proxpero/189a723fb96bb88fac5bf9e11d6cf9e2#gistcomment-2148781">significant improvements</a> suggested by <a href="https://twitter.com/stephencelis">@stephencelis</a>.
---
One way to allow an enumeration with associated values to conform to the `Codable` protocol is to represent it in a struct which *is* `Codable`. Another, better way is to use the `Codable` container provided by Apple. I'm going to show how to do both step-by-step, using the `Barcode` example from [the Enumeration section](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Enumerations.html#//apple_ref/doc/uid/TP40014097-CH12-ID145) of the [Swift Programming book](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/). (I won't make the joke that you should stop reading this to go read that while I wait for you, I'm just assuming you've already done it. Surely you've also read about [the `Codable` protocol](https://developer.apple.com/documentation/swift/encoding_decoding_and_serialization) as well.) My hope is that, by including an explanation of an inferior method, some insight will be gained as to what framework is doing (more or less) under the hood.

Some `enum`s come prepopulated with raw values, like for example this one in which each planet case is assigned an `Int` value.
{% highlight swift %}
enum Planet: Int {
    case mercury = 1, venus, earth, mars, jupiter, saturn, uranus, neptune
}
{% endhighlight %}
In this case, archiving the enum instance is as easy as archiving an integer. But what about those enum types with associated values?

{% include toc.html %}

## Barcode

Apple's `Barcode` implementation is simple:

{% highlight swift %}
enum Barcode {
    case upc(Int, Int, Int, Int)
    case qrCode(String)
}
{% endhighlight %}

This enum however contains more information than can be represented by a simple raw value type like a `String` or an `Int` and so it cannot conform to `Codable` without additional instructions as to how each case is to be coded. One way to provide this information is to create an intermediate type that knows how to do just that.

(By the way, these instructions, however they are provided, are not something we ought to expose to the wider world. Ideally, all that users need to know is that `Barcode` is `Codable`, so let's keep those implementation details private, ok?)

## The Intermediate Type

I'm going to start by declaring a struct called `Coding`, nested inside an extension of `Barcode` and conforming to `Codable`. It's not `Codable` yet, but we'll get it there. This type needs to be able to represent all the information that is possible to represent in the original `Barcode` enum, and so presumably it will need a different property for each case, where the type of the property matches the type of its case's associated value.
{% highlight swift %}
extension Barcode {
    fileprivate struct Coding: Codable {
        private var upc: (Int, Int, Int, Int)
        private var qrCode: String
    }
}
{% endhighlight %}
But this doesn't quite work. The point of the intermediate type is to create a representation of `Barcode` which is `Codable`, but the type of `upc` is `(Int, Int, Int, Int)` which is a tuple and tuples are not `Codable`.

One workaround for this is to represent the tuple as an `Array<Int>` since collections of `Codable` types are themselves `Codable` ([tuples are not collections](https://stackoverflow.com/questions/34847699/why-isnt-a-swift-tuple-considered-a-collection-type#34848318), nor should they be), but I'm going to create another, separate intermediate type to represent `(Int, Int, Int, Int)`: a struct with four `Int` properties to represent each `Int` of the `upc` value, and *then* we'll be back in business.
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

To do this requires one more tweak to our intermediate type. If there's one thing an enumeration is good at, it's this: An instance is one case *or* another, but *never* both, guaranteed. We ought to include this feature, insofar as we can, in our `Barcode.Coding` type, since we're trying to represent an enum. The natural way to do this is to have its properties be optional so that when a `Barcode` instance is a `.upc`, then the `qrCode` property of its `Barcode.Coding` struct can be `nil`, and vice versa.

We'll write an explicit `init` method in the body of the struct (not in an extension because we want to replace the memberwise initializer in order to narrow the ways this struct can be created) and this `init` method will guarantee that exactly one of the properties can ever receive a value and all other properties will remain `nil`. This guarantee will be provided by switching on the original enum.

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

We now have the ability to convert a `Barcode` to a `Barcode.Coding` type which is `Codable`, we now need implement the conversion in the other direction. We just need a method on `Barcode.Coding` that returns a `Barcode`. We'll make it `throw` for reasons that will become clear in a moment.
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

After all that work, we're finally ready to write the code that will conform our `Barcode` type to `Codable`. First, we only need to implement two methods, one each for `Encodable` and `Decodable`, and second, we need to explicitly declare conformance.
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

## Informal Testing

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

> **Protip**: We try to be aesthetic and all here on proxpero.com. But seriously, don't use pretty-printing for testing JSON equality. Also, until `Equatable` is actually implemented on `Barcode`, the `==` operator is not going to work. I'm going to leave that as an exercise for you.

## But Does It Scale?

Even with only two cases, this was not a trivial amount of code. It's not difficult; in fact, it's very like boilerplate. With larger numbers of cases, writing all the tedious code required to get an enumeration with associated values to be `Codable` will make you ask yourself: What am I getting for this work?

For one thing, it is likely that not all of your project's types are enums with associated values. Many will easily conform to `Codable` and you will want all your types to conform if you want any of them to. And you probably do want them to, since (1) it is in your long term interest to adopt fundamental Apple technologies in your Apple platform projects. And (2) I think Apple has  done a good job with this protocol and it is worthy to become widely adopted.

Perhaps future versions of Swift with more robust reflection abilities will make archiving workarounds like this one obsolete. I certainly would welcome that day. But until that version lands, it was right Apple to provide the limited implementation that it did now, rather than wait until it could do it "the right way". Real artists ship.

## Improvements

Even with only two cases, this was not a trivial amount of code. It's not difficult; in fact, it's very like boilerplate. With larger numbers of cases, writing all the tedious code required to get an enumeration with associated values to be `Codable` will make you ask yourself: What am I really getting for all this typing? Turns out, there's a better solution. (Special thanks to [@stephencelis](https://twitter.com/stephencelis) for [showing this](https://gist.github.com/proxpero/189a723fb96bb88fac5bf9e11d6cf9e2#gistcomment-2148781) to me.)

One of my favorite Stack Overflow answers (I can't find the link) was a response to a question about how many lines of codes to write in one day. The answer was something like
> On good days, it's measured in negative numbers. On better days, in higher negative numbers.

Smaller units are more easily understandable, less code means fewer bugs. But often we can't start with the smaller codebase. The first priority is to get something working, and iterate afterwards, reducing our material until we've reached the bare necessities. One consequence of this approach is that the bloated, middle period nevertheless provides intuition about the workings of the final result.

Our first solution used an intermediate, `Codable` type that we implemented ourselves. As it happens, the `Encoder` and `Decoder` protocols already provide a struct for the same purpose. Some configuration is still required but at least we don't need to create our own `Coding` struct. Let's start over.

## The Keyed(En|De)codingContainer

The `Encoder` protocol specifies only a handful of requirements. One is that a conforming type implement this function:

{% highlight swift %}
func container<Key>(keyedBy type: Key.Type) -> KeyedEncodingContainer<Key>
{% endhighlight %}

It is a function with a generic type parameter and it returns a container which is generic over that same type. This container, an instance of `KeyedEncodingContainer`, serves the same purpose as our `Coding` struct from earlier. Looking at the return type of this function, we see that `KeyedEncodingContainer` is generic over some `Key`, looking at the definition of this type, we see that this generic parameter must be constrained to the `CodingKey` protocol.
{% highlight swift %}
/// A concrete container that provides a view into an encoder's storage, making
/// the encoded properties of an encodable type accessible by keys.
public struct KeyedEncodingContainer<K> : KeyedEncodingContainerProtocol where K : CodingKey {
  /// Implementation details...
}
{% endhighlight %}

The `CodingKey` protocol provides a mapping from the names of the properties of the encoded type to the keys of the underlying storage.

{% highlight swift %}
enum Barcode {
    case upc(Int, Int, Int, Int)
    case qrCode(String)
}

extension Barcode: Codable {
    private enum CodingKeys: String, CodingKey {
        case upc
        case qrCode
    }
}
{% endhighlight %}

We've specified a private enum called `CodingKeys`. (The name can be anything, but "CodingKeys" follows a convention established by Apple. Note that it is plural whereas usually enum names are singular.) The enum provide raw values for its cases, either `String`s or `Int`s and must declare conformance to the `CodingKey` protocol. For convenience I've chosen cases that match the property names of `Barcode`, but again these are arbitrary, additionally you could even specify a raw value different from the case. Now we can implement the two methods needed to conform to `Codable`. We'll start with encoding.

{% highlight swift %}
func encode(to encoder: Encoder) throws {
    var container = encoder.container(keyedBy: CodingKeys.self)
    switch self {
    case .upc(let numberSystem, let manufacturer, let product, let check):
        try container.encode([numberSystem, manufacturer, product, check], forKey: .upc)
    case .qrCode(let productCode):
        try container.encode(productCode, forKey: .qrCode)
    }
}
{% endhighlight %}

To begin with, we get the `KeyedEncodingContainer` from `encoder`. Then, we switch on `self`. If `self` is a `Barcode.upc`, we get the four `Int` values out, and then we'll try to encode an array of the four upc values in the container and the key for that value in the container will be the rawValue of `Barcode.CodingKeys.upc`. (I made an intermediate type to store the four upc values in the earlier implementation. This time I'm just using an array.) Likewise, if `self` is a `Barcode.qrCode`, we get the `String` value, and then try to encode it into the `container` using the key `Barcode.CodingKeys.qrCode`.

> **Warning**: I've used `.upc` in two different places in the code above. The compiler knows they are different, but I wanted to explicitly point it out. One is a `Barcode.upc` and the other is a `Barcode.CodingKey.upc`. The same is true of the `.qrCode`.

The `Decoder` protocol similarly provides a `KeyedDecodingContainer`, which we can use in the init method.

{% highlight swift %}
enum CodingError: Error {
    case decoding(String)
}

init(from decoder: Decoder) throws {
    let container = try decoder.container(keyedBy: CodingKeys.self)
    if let codes = try? container.decode(Array<Int>.self, forKey: .upc), codes.count == 4 {
        self = .upc(codes[0], codes[1], codes[2], codes[3])
        return
    }

    if let code = try? container.decode(String.self, forKey: .qrCode) {
        self = .qrCode(code)
        return
    }

    throw CodingError.decoding("Decoding Error: \(dump(container))")
}
{% endhighlight %}

Again, we get the container first and then try to extract the appropriate values from it. If we can assign to `codes` an array of integers using the key `Barcode.CodingKeys.upc` (actually, the `rawValue` of it), then we will initialize a `Barcode.upc` whose associated values map to the four elements of the `Array<Int>` that we extracted. If that doesn't work, we try to assign to `code` a `String` using as the key `Barcode.CodingKeys.qrCode` (again it is really `rawValue` of the `.qrCode` coding key) and use that `String` to create the corresponding `Barcode.qrCode` and assign it to `self`. If neither of those works then we'll throw a custom error.

Here's the full solution:

{% highlight swift %}
enum Barcode {
    case upc(Int, Int, Int, Int)
    case qrCode(String)
}
extension Barcode: Codable {
    private enum CodingKeys: String, CodingKey {
        case upc
        case qrCode
    }
    enum CodingError: Error {
        case decoding(String)
    }
    init(from decoder: Decoder) throws {
        let values = try decoder.container(keyedBy: CodingKeys.self)
        if let codes = try? values.decode(Array<Int>.self, forKey: .upc), codes.count == 4 {
            self = .upc(codes[0], codes[1], codes[2], codes[3])
            return
        }
        if let code = try? values.decode(String.self, forKey: .qrCode) {
            self = .qrCode(code)
            return
        }
        throw CodingError.decoding("Decoding Error: \(dump(values))")
    }
    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        switch self {
        case .upc(let numberSystem, let manufacturer, let product, let check):
            try container.encode([numberSystem, manufacturer, product, check], forKey: .upc)
        case .qrCode(let productCode):
            try container.encode(productCode, forKey: .qrCode)
        }
    }
}

import Foundation

let encoder = JSONEncoder()
encoder.outputFormatting = .prettyPrinted
let decoder = JSONDecoder()
var productBarcode: Barcode
var data: Data
var json: String
var expectation: String
var result: Barcode

productBarcode = .upc(8, 85909, 51226, 3)
data = try encoder.encode(productBarcode)
json = String.init(data: data, encoding: .utf8)!
expectation = """
{
  "upc" : [
    8,
    85909,
    51226,
    3
  ]
}
"""
print(json)
result = try decoder.decode(Barcode.self, from: data)
//assert(result == productBarcode)


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
////assert(result == productBarcode)
{% endhighlight %}

## Conclusion

I'm pretty happy with Apple's solution to the problem of archiving Swift value types. The fact that it doesn't just work in every case is not a reason not to use it. Some assembly is required. Using Apple's `KeyedDecoderContainer` and `KeyedEncoderContainer` types helps minimize the burden on developers who need to serialize enums with associated values.

## References

- This code is available as a [gist](https://gist.github.com/proxpero/189a723fb96bb88fac5bf9e11d6cf9e2). You can paste it into a playground.
- Apple has a [good article](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types) documenting `Codable`s behavior.
- The [proposal](https://github.com/apple/swift-evolution/blob/master/proposals/0166-swift-archival-serialization.md) on Swift Evolution.
- The [actual implementation of `Codable`](https://github.com/apple/swift/blob/master/stdlib/public/core/Codable.swift) in the Swift repo.
- The [actual implementation of `JSONEncoder`](https://github.com/apple/swift/blob/master/stdlib/public/SDK/Foundation/JSONEncoder.swift) in the Swift repo.
- Greg Heo wrote [a detailed investigation](https://swiftunboxed.com/stdlib/json-encoder-encodable/) of the JSONEncoder.

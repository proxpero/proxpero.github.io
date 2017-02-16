---
title: An Implementation of the MD5 Hash Algorithm
summary: "I present a small, zero-dependency extension that computes the md5 hash function and does nothing else."
tags: Swift md5
---
[The MD5 message digest algorithm](https://tools.ietf.org/html/rfc1321) is a [hash function](https://en.wikipedia.org/wiki/Hash_function) that was designed for cryptography but is [no longer considered safe](https://en.wikipedia.org/wiki/MD5#Security) for [that purpose](https://www.quora.com/Cryptography-Why-are-MD5-and-SHA1-called-broken-algorithms/answer/Michael-Hamburg). This is not to say that it isn't still [useful](http://khanlou.com/2015/07/cache-me-if-you-can) for [other things](http://adventofcode.com/2016/day/17). It still can take a chunk of data of arbitrary size (it could be 0x1, or it could be the text of [*Moby Dick*](http://www.clickhole.com/blogpost/time-i-spent-commercial-whaling-ship-totally-chang-768)) and transform it into a fixed 128-bit chunk of data that appears to be random garbage, but in fact the same input to the function will always produce the same output. Often, you just want the hash of some string or some data and you don't need or want to [import](http://stackoverflow.com/questions/25248598/importing-commoncrypto-in-a-swift-framework) a whole [crypto library](https://github.com/krzyzanowskim/cryptoswift) for such a limited, trivial use.
With that in mind, I'm presenting here my own implementation ([available as a gist](https://gist.github.com/proxpero/c5aedb64f19f0d4c9ae63b8eee1a1b3e)) of the **md5 hash function** in one small (180 lines including white space and tests) file with no dependencies (not even `Foundation`) comprising two extensions.

The first is an extension on `String`, which allows you to do things like

{% highlight Swift %}
let hash = "do not alter this string: I will know if you do!".md5
{% endhighlight %}

It looks like this:

{% highlight Swift %}
extension String {
    public var md5: String {
        return self.utf8.md5
    }
}

{% endhighlight %}

Not much there, but it's worth noting that [under the hood](https://developer.apple.com/reference/swift/string.utf8view) `String.UTF8View` is a collection of `UInt8`s, and therefore the `String` extension can easily invoke the logic contained in the collection extension.

So, this other extension on `Collection` is constrained to those where `Iterator.Element` is `UInt8`. This includes not only `String.UTF8View` but also `Foundation`'s `Data` type, which allows things like this:

{% highlight Swift %}
import Foundation // required for `Data`
let sketchyApp = Data(from: internet)
let hash = sketchyApp.md5
{% endhighlight %}

The `md5` method (really a "computed property") on the collection is slightly more interesting. It lazily reduces the array of bytes into a single hexadecimal string.

{% highlight Swift %}
extension Collection where Iterator.Element == UInt8 {
    public var md5: String {
        return self.md5Digest.lazy.reduce("") {
            var s = String($1, radix: 16)
            if s.characters.count == 1 {
                s = "0" + s
            }
            return $0 + s
        }
    }
}
{% endhighlight %}

The real logic is contained in the private `md5Digest` computed property of the collection. It takes the bytes in the collection and returns the array of bytes that make up the hash. It is this result that is then turned into a `String` in the `md5` computed property above.

I'm not going to go into point-by-point detail, but the code closely follows the explanation of md5 presented in [RFC 1321](https://tools.ietf.org/html/rfc1321) by [Ronald Rivest](https://www.youtube.com/watch?v=YQw124CtvO0). Within the code, I've interspersed snippets of documentation from [Wikipedia](https://en.wikipedia.org/wiki/MD5#Pseudocode) and [RFC 1321](https://tools.ietf.org/html/rfc1321) as signposts for the curious to follow. *Note:* I have **intentionally** leaned away from writing good, idiomatic, designed-for-clarity Swift by retaining the historical names for variables and functions, which are the heritage of a different era. If you want to understand this code, you will probably want to compare it to [some](https://www.quora.com/How-does-the-MD5-algorithm-work)  [descriptions](http://www.iusmentis.com/technology/hashfunctions/md5/) of the algorithm, and these will likely use the same names as the original. This code is not necessarily intended to [pass a review](http://stackoverflow.com/questions/109023/how-to-count-the-number-of-set-bits-in-a-32-bit-integer#comment1668520_109025). 😉

{% highlight Swift %}
extension Collection where Iterator.Element == UInt8 {
  private var md5Digest: [UInt8] {
      typealias Word = UInt32
      typealias Byte = UInt8
      // s specifies the per-round shift amounts
      let s: [Word] = [
          7, 12, 17, 22, 7, 12, 17, 22, 7, 12, 17, 22, 7, 12, 17, 22,
          5,  9, 14, 20, 5,  9, 14, 20, 5,  9, 14, 20, 5,  9, 14, 20,
          4, 11, 16, 23, 4, 11, 16, 23, 4, 11, 16, 23, 4, 11, 16, 23,
          6, 10, 15, 21, 6, 10, 15, 21, 6, 10, 15, 21, 6, 10, 15, 21
      ]
      // K[i] = abs(sin(i + 1)) * 232 (radians)
      let k: [Word] = [
          0xd76aa478, 0xe8c7b756, 0x242070db, 0xc1bdceee,
          0xf57c0faf, 0x4787c62a, 0xa8304613, 0xfd469501,
          0x698098d8, 0x8b44f7af, 0xffff5bb1, 0x895cd7be,
          0x6b901122, 0xfd987193, 0xa679438e, 0x49b40821,
          0xf61e2562, 0xc040b340, 0x265e5a51, 0xe9b6c7aa,
          0xd62f105d, 0x02441453, 0xd8a1e681, 0xe7d3fbc8,
          0x21e1cde6, 0xc33707d6, 0xf4d50d87, 0x455a14ed,
          0xa9e3e905, 0xfcefa3f8, 0x676f02d9, 0x8d2a4c8a,
          0xfffa3942, 0x8771f681, 0x6d9d6122, 0xfde5380c,
          0xa4beea44, 0x4bdecfa9, 0xf6bb4b60, 0xbebfbc70,
          0x289b7ec6, 0xeaa127fa, 0xd4ef3085, 0x04881d05,
          0xd9d4d039, 0xe6db99e5, 0x1fa27cf8, 0xc4ac5665,
          0xf4292244, 0x432aff97, 0xab9423a7, 0xfc93a039,
          0x655b59c3, 0x8f0ccc92, 0xffeff47d, 0x85845dd1,
          0x6fa87e4f, 0xfe2ce6e0, 0xa3014314, 0x4e0811a1,
          0xf7537e82, 0xbd3af235, 0x2ad7d2bb, 0xeb86d391
      ]
      // Initialize variables
      var a: Word = 0x67452301
      var b: Word = 0xefcdab89
      var c: Word = 0x98badcfe
      var d: Word = 0x10325476
      var message = Array(self)
      // The original length of the message in bits (before padding)
      let length = UInt64(message.count * 8)
      // Pre-processing: adding a single 1 bit
      message.append(0x80)
      // append "0" bit until message length in bits ≡ 448 (mod 512)
      repeat {
          message.append(0x0)
      } while (message.count * 8) % 512 != 448
      // A 64-bit representation of `length` is appended to the result of the previous step.
      for i in (0...7).map({ UInt64($0 * 8) }) {
          message.append(Byte((length >> i) & 0xFF))
      }
      // Divide message into successive 512-bit (64-byte) chunks
      let chunks = (0..<(message.count / 64))
          .map { $0 * 64 }
          .map { Array(message[$0..<$0+64]) }
      // Process each chunk:
      for chunk in chunks {
          // Break chunk into sixteen 32-bit words m[i], 0 ≤ i ≤ 15
          func part(_ index: Int, _ offset: Int, _ shift: Word) -> Word {
              return Word(chunk[4 * index + offset]) << shift
          }
          var m = (0...15).map { i in
              part(i, 0, 0) | part(i, 1, 8) | part(i, 2, 16) | part(i, 3, 24)
          }
          // Initialize hash value for this chunk:
          var A = a
          var B = b
          var C = c
          var D = d
          //Main loop:
          for index in k.indices {
              var f: Word = 0
              var g = 0
              switch index {
              case 0...15:
                  f = (B & C) | ((~B) & D)
                  g = index
              case 16...31:
                  f = (B & D) | (C & (~D))
                  g = ((5*index + 1) % 16)
              case 32...47:
                  f = B ^ C ^ D
                  g = ((3*index + 5) % 16)
              case 48...63:
                  f = C ^ (B | (~D))
                  g = ((7*index) % 16)
              default:
                  break
              }
              func rotateLeft(_ x: Word, by amount: Word) -> Word{
                  return ((x << amount) & 0xFFFFFFFF) | (x >> (32 - amount))
              }
              let dTemp = D
              D = C
              C = B
              B = B &+ rotateLeft(A &+ f &+ k[index] &+ m[g], by: s[index])
              A = dTemp
          }
          // Add this chunk's hash to result so far:
          a = a &+ A
          b = b &+ B
          c = c &+ C
          d = d &+ D
      }
      // a append b append c append d
      let result = [a, b, c, d].flatMap { word in
          [Word]([00, 08, 16, 24]).map { Byte((word >> $0) & 0xFF) }
      }
      return result
  }
}
{% endhighlight %}
Needless to say, this implementation passes the original test suite provided by RFC 1321.
{% highlight Swift %}
let tests = [
    ("", "d41d8cd98f00b204e9800998ecf8427e"),
    ("a", "0cc175b9c0f1b6a831c399e269772661"),
    ("abc", "900150983cd24fb0d6963f7d28e17f72"),
    ("message digest", "f96b697d7cb7938d525a2f31aaf161d0"),
    ("abcdefghijklmnopqrstuvwxyz", "c3fcd3d76192e4007dfb496cca67e13b"),
    ("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789", "d174ab98d277d9f5a5611c2c9f419d9f"),
    ("12345678901234567890123456789012345678901234567890123456789012345678901234567890", "57edf4a22be3c955ac49da2e2107b67a")
]
for (text, hash) in tests {
    assert(text.md5 == hash)
    print("test passed.")
}
{% endhighlight %}

And that's all there is to it! Thanks for reading!

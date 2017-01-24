---
layout: post
title: The Beginning of the Endgame
comments: false
mathjax: false
ganalytics: true
---

[Last time], I introduced my chess framework [Endgame] and gave a brief overview of its structure. Now, I'm going to explain more about the lowest of the four main components, the bitboard.

[Last time]: https://proxpero.com/2017-1-17-Endgame-part1-Introduction
[Endgame]: https://github.com/proxpero/Winchester/tree/master/Endgame

### Bitboard

A `bitboard` is just a wrapper around a `UInt64`, a dandy type for us since it stores exactly 64 bits and there just happen to be exactly 64 squares on a chessboard. This means that many computed properties necessary for our purposes can be provided merely by clever and inexpensive bitwise arithmetic.

```Swift
public struct Bitboard: RawRepresentable {

  public var rawValue: UInt64

  public init(rawValue: UInt64) {
    self.rawValue = rawValue
  }

}
```

More specifically, a large class of queries about the state of the board can be answered by a simple number.

> Q: What are all the squares on the board?

Easy. It's 18446744073709551615. That's the number whose binary representation (in an 8x8 grid) is
```
11111111
11111111
11111111
11111111
11111111
11111111
11111111
11111111
```

> Q: Which squares are in the 6th rank?

Simple. 280375465082880.
```Swift
00000000
00000000
11111111
00000000
00000000
00000000
00000000
00000000
```

> Q: What are the starting positions of the white knights?

66.
```
00000000
00000000
00000000
00000000
00000000
00000000
00000000
01000010
```

> Q: Square `c3`?

262144.

```
00000000
00000000
00000000
00000000
00000000
00000100
00000000
00000000
```

> Q: Wait that looks like the `f3` square. What's up?

That's because I'm writing the binary numbers with the least significant bit on the right, with padded zeros to the left. For example, the square `a1` has a bitboard value of `1` and it occupies the lower lefthand side of the board. But in the grid of binary numbers above the number one would be placed in the lower righthand corner. The point is not to show that numbers make a good human-readable UI, but that real chess situations can be modeled with simple numbers.

Later, I'll show you how to make more complex queries like "What are the possible destinations of a bishop on the `e4`?" by conforming `Bitboard` to the `BitwiseOperations` protocol.

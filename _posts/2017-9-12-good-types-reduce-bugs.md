---
title: "Good Types Reduce Bugs"
summary: "The advantages of working in a type-safe language extend beyond the compiler complaining that it received an <code>Double</code> but expected an <code>Int</code>. Developers can use the Swift type system to enforce complex invariants rather than relying on mere conventions or other fragile means, but reaping the benefits requires some forethought and of bit careful design. I use an example modeling a network connection to illustrate this idea."
tags: [Swift, Type Safety]
toc:
  - title: A Naive Approach
    link: /#a-naive-approach
  - title: A Better Solution
    link: /#a-better-solution
  - title: Summary
    link: /#summary
---
It is largely accepted in the Swift community that the type system reduces bugs. There are trivial examples of the compiler catching mistakes such as passing the wrong numeric type to function. This class of bug is found with no help needed from developer. True enough. But when the developer does make the effort to carefully design types, then the more profound benefits of strong, static typing become available.

{% include toc.html %}

## A Naive Approach

Here's one way to represent the state of a network connection.

{% highlight swift %}
enum ConnectionState {
    case connecting
    case connected
    case disconnected
}
struct ConnectionInfo {
    let state: ConnectionState
    let server: IPAddress
    let lastPingTime: Date?
    let lastPingId: Int?
    let sessionId: String?
    let initiated: Date?
    let disconnected: Date?
}
{% endhighlight %}

The `ConnectionState` is a simple enum with three possible cases. The `ConnectionInfo` struct provides information about different aspects of a connection. The `state` and the `server` fields are required, the rest are optional. The optional fields are not populated willy-nilly: they operate under strict invariants:

  -  `lastPingTime` and `lastPingId` are used to keep the connection alive. They are optional because **they should only be present when `state` is `connected`, and in that case both must be present.**
  - `sessionId` is a unique identifier for each new connection and **it should only be present when `state` is `connected`.**
  - `initiated` is a timestamp used to determine whether the attempt to connect should continue. **It should only be present when `state` is `connecting`.**
  - Similarly, `disconnected` is a timestamp but is used to record when the connect disconnected. **It should only be present when `state` is `disconnected`.**

There are numerous ways this code can break. Preventing that requires precise documentation. Each client must be subjected to complex tests to confirm it is using this code correctly. Therefore, it requires *continuous vigilance* on the part of developers to maintain these invariants.

## A Better Solution

What we want is to be able to write the code once and forget it, confident that it works and cannot be abused. That's the dream anyway, and we can get close if we think carefully about our types. As it turns out, these invariants can be baked directly into the code. Then, you don't need to document how the code works, that would be like explaining how Swift itself works. It doesn't need complex tests, because you don't need to test Swift.

{% highlight swift %}

struct Ping {
    let time: Date
    let id: Int
}
enum ConnectionState {
    case connecting(initiated: Date)
    case connected(lastPing: Ping?, sessionId: String)
    case disconnected(disconnected: Date)
}
struct ConnectionInfo {
    let state: ConnectionState
    let server: IPAddress
}

{% endhighlight %}

`ConnectionInfo` has simplified down to its bare requirements: it must have a `state` and an `server` address. The previous optional fields have been moved to associated types on their appropriate cases.

  - When the `state` is `connecting`, there **must** be a date stored right there in the case.
  - When the `state` is `connected`, there **must** be a `sessionId` instance available just the to `connected` instance. There **may** be an instance of `Ping` as well, only if any pings were sent. But you know that `lastPing`, if it is not nil, will have both a `time` and an `id`.
  - When `state` is `disconnected`, you can depend on the `disconnected` timestamp to be available.

## Summary

Now that the invariants are part of the types themselves, the compiler can detect and reject code that violates these invariants. This is less work and more reliable than trying to maintain these invariants by hand. Both examples are valid Swift, but the second is superior because it shifts more overhead from the developer to the language.

## Further Reading

  - [Enumerations](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Enumerations.html#//apple_ref/doc/uid/TP40014097-CH12-ID145) See the part especially about Associated Values.
  - [OCaml for the Masses](https://cacm.acm.org/magazines/2011/11/138203-ocaml-for-the-masses/fulltext) (...where I stole my example. There is much other insightful information here too. It is about OCaml but is nevertheless relevant to Swift.)
  - [Serializing Enums with Associated Values](http://proxpero.com/2017/07/11/encoding-and-decoding-custom-enums-with-associated-values-in-swift-4/) (Shameless!)

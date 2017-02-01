---
title: Generating Text with Markoff Chains
summary: I show how to produce superficially real-looking text using a Markoff chain.
tags: [Swift, Markoff Chains, Sequence protocol]
---
In this post I'm going to show how you can create [superficially real-looking text] using a [Markoff chain]. The idea is that as you read through some body of text, or *corpus*, you record a chain of words called a prefix and the word following the chain called the suffix. The length of the prefix is some arbitrary number, say two or three. If your corpus is large enough, you'll have several one-word suffixes associated with every multiple-word suffix.

[superficially real-looking text]: https://en.wikipedia.org/wiki/Natural_language_generation
[Markoff chain]: https://en.wikipedia.org/wiki/Markov_chain

For example, in the course of reading the corpus, you might have come across the the following prefix two times: "he died in". The first time it occurred, it was followed by the word "battle", the second time by the word "vain". The implementation must record the prefix and the two associated suffixes, perhaps in a dictionary where the key is the prefix and the value an array of suffixes.

## The `Chain` struct

Let's start with a struct that will model our chain.

> Note that we need to import the `Foundation` since we are using `CharacterSet`.

This is more text.

{% highlight swift %}

public struct Chain {
    private var store: Dictionary<String, [String]> = [:]
    init(text: String, prefixLength: Int) {



    }
  }

{% endhighlight %}

The `Chain` struct is initialized with some text and prefix length and builds the chain by going through the text word by word, and the result is stored in the private `store` dictionary.

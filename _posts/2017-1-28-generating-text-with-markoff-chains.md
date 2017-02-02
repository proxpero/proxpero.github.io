---
title: Generating Text with Markoff Chains
summary: I show how to produce superficially real-looking text using a Markoff chain.
tags: [Swift, Markoff Chains, Sequence protocol]
---
In this post I'm going to show how you can create [superficially real-looking text] using a [Markoff chain]. The idea is that as you read through some body of text, or *corpus*, you record a chain of words called a prefix and the word following the chain called the suffix. The length of the prefix is some arbitrary number, say two or three. If your corpus is large enough, you'll have several one-word suffixes associated with every multiple-word suffix.

[superficially real-looking text]: https://en.wikipedia.org/wiki/Natural_language_generation
[Markoff chain]: https://en.wikipedia.org/wiki/Markov_chain

For example, in the course of reading the corpus, you might have come across the the following prefix two times: "he died in". The first time it occurred, it was followed by the word "battle", the second time by the word "vain". The implementation must record the prefix and the two associated suffixes, perhaps in a dictionary where the key is the prefix and the value an array of suffixes.

## Initializing the `Chain` struct

Our `Chain` struct will have a single stored property, a private `Dictionary`. I'll call it "store". The keys will be prefixes and the value for each key will be all suffixes associated for that prefix.

{% highlight swift %}

public struct Chain {
    private var store: Dictionary<String, [String]> = [:]
    // TODO: a public initializer goes here.
}
{% endhighlight %}

The initializer takes some text and a prefix length, building the chain by scanning through the text, word by word. The new prefix will be built by taking the previously constructed prefix, appending the newly encountered word, and dropping the oldest word. Think of a window sliding along a line of text. As the left most word disappears from view, a new word enters the frame on the right, and that line of visible words forms the new prefix. The number of words exposed by the window is determined by the `prefixLength` parameter.

As the constructor builds the prefix, is also determines the suffix of that prefix. Think of a second window adjacent to the prefix window, sliding along with it on the right, exposing only one word. The suffix is appended to the array of suffixes already found for that prefix, or to an empty array if it's the first one.

{% highlight swift %}
public init(text: String, prefixLength: Int) {
    var result: Dictionary<String, [String]> = [:]
    // TODO: Nest the implementation of `prefixIterator` here.
    for (prefix, suffix) in AnySequence(prefixIterator) {
        var suffixes = result[prefix] ?? [String]()
        suffixes.append(suffix)
        result[prefix] = suffixes
    }
    self.store = result
}
{% endhighlight %}

There's our implementation. We loop through a sequence of tuples consisting of prefix/suffix pairs and steadily build up our chain with new links. But where does the sequence come from?

`AnySequence` is [a type-erased sequence], a struct that conforms to the `Sequence` protocol. One of its initializers takes a function that returns an iterator. `prefixIterator` in our implementation above *is that function*.

> See [here][A], [here][B], and [here][C] for more information about making custom Sequences and Iterators in Swift. See [here][0], [here][1], [here][2], and [here][3] for more on type erasure. These aren't simple topics to understand, especially if you're coming from Objective-C.

[a type-erased sequence]: https://developer.apple.com/reference/swift/anysequence
[A]: https://www.uraimo.com/2015/11/12/experimenting-with-swift-2-sequencetype-generatortype/
[B]: http://codereview.stackexchange.com/a/136101
[C]: https://www.objc.io/books/advanced-swift/
[0]: http://www.russbishop.net/inception
[1]: https://realm.io/news/tryswift-gwendolyn-weston-type-erasure/
[2]: https://krakendev.io/blog/generic-protocols-and-their-shortcomings
[3]: http://robnapier.net/erasure

Here's our implementation.

{% highlight swift %}
func prefixIterator() -> AnyIterator<(String, String)> {
    let words = text.words
    var index: Int = 0
    return AnyIterator {
        defer {
            index += 1
        }
        let n = index > prefixLength ? prefixLength : index + 1
        let prefix = words.dropFirst(index).prefix(n).joined(separator: " ")
        guard let suffix = words.dropFirst(index + n).first else { return nil }
        return (prefix, suffix)
    }
}
{% endhighlight %}

Our function returns an `AnyIterator`, a [type-erased iterator], that is generic over a tuple of type `(String, String)`, which indicates to the complier the type of the elements of our iterator. As you may have guessed, these two strings will be the prefix/suffix pairs we were using to build our chain earlier. Just as `AnySequence` is a struct conforming to the `Sequence` protocol, `AnyIterator` is a struct that conforms to `IteratorProtocol`, and its initializer also takes a function. This function basically provides the `IteratorProtocol`'s one required method: `next() -> Element?`, returning the next element in the sequence, or else `nil` if the sequence has no more elements.

[type-erased iterator]: https://developer.apple.com/reference/swift/anyiterator

To repeat, we return an `AnyIterator`, which is initialized with a function with the signature `() -> (String, String)?`. And this function is invoked every time the sequence needs the next element. Since the parameter of `AnyIterator.init` is itself a function, I can use [trailing closure syntax](https://www.youtube.com/watch?v=PcwrOmygzgY) and dispense with the parentheses. Note that the mutating state variable `index`, the only thing that ever changes while the sequence is looping, is declared outside the scope of the closure but is captured inside the closure, so it's memory will be preserved across invocations and each invocation of the closure can read and mutate the changing values of `index`.

As for the logic inside, `index` keeps track of the current location in the array of words. Then, starting at that location, we take however many words we need, based on the length of the prefix. We join these words together with spaces to make a single string, that's our prefix. The next word after that is our suffix. Return the pair as a tuple, but before closure is completely finished, it increments `index` by one.

> `words` is our corpus text split on newlines and whitespaces. It would have been more space efficient to implement this function by finding the ranges of the word boundaries in the `text` string itself and returning substrings, instead of creating a superfluous array. I'll revisit this issue [when Strings are fixed in Swift 4](https://github.com/apple/swift/blob/master/docs/StringManifesto.md). 😉

The `Chain` struct so far looks like this.

{% highlight swift %}
///
public struct Chain {
    private var store: Dictionary<String, [String]> = [:]

    ///
    public init(text: String, prefixLength: Int) {
        var result: Dictionary<String, [String]> = [:]
        func prefixIterator() -> AnyIterator<(String, String)> {
            let words = text.words
            var index: Int = 0
            return AnyIterator {
                defer { index += 1 }
                let n = index > prefixLength ? prefixLength : index + 1
                let prefix = words.dropFirst(index).prefix(n).joined(separator: " ")
                guard let suffix = words.dropFirst(index + n).first else { return nil }
                return (prefix, suffix)
            }
        }

        for (prefix, word) in AnySequence(prefixIterator) {
            var current = result[prefix] ?? [String]()
            current.append(word)
            result[prefix] = current
        }
        self.store = result
    }
}
{% endhighlight %}

## Generating (Fake) Text

Now that we have a `Chain`, we can use to start producing our "superficially real-looking text". Let's start with a public method.

{% highlight swift %}
public func generateText(maxWordCount: Int) -> String {
    var result = ""
    // TODO: Turn `result` into something interesting.
    return result
}
{% endhighlight %}

There's not much going on with this method yet. It takes an integer which basically limits the amount of text produced. And, predictably, it has a return type `String`. I start with an empty string called `result`, which will get mutated somehow during the course of the function by appending to it one word at a time until we've reached the limit. But, in order to know what words should be added, we need to use the `store` dictionary we created in the initializer.

We need to start with a random prefix. All the subsequent text will grow from this seed. And since we want our text to sound like complete sentences, let's make sure we get a prefix that's suitable for opening a sentence. We'll say it's good enough for now to choose only a prefix that begins with an uppercase letter.

{% highlight swift %}
var seed = ""
while !seed.hasInitialCapital {
    seed = store.randomKey
}
{% endhighlight %}

I'm using some custom extensions here. The computed property `hasInitialCapital` is an extension on the `String` type that returns true if the first character of the string is is an uppercase letter. `randomKey` comes from an extension on `Dictionary` where a key chosen at random and returned.

Now that we have a starting point, we can set the sequence going. Here's what we're going to do. We'll take our prefix and look it up in our `store` dictionary. That will give us the possible suffixes. We'll chose one at random and append it to result. Having chosen a suffix we need also to use it to create the next prefix in the sequence. We'll create it by dropping the first word of the current prefix and appending the chosen suffix to the end, making a new string. This new prefix will be the new prefix key that we look up in `store` to find the next suffix.

We'll use a `Sequence` type again, but this time we'll use the free function `sequence(state:next:)`. We'll only have to provide the "state", which in our case is the seed we created earlier, and also a function which returns the next element, which in our case is the new suffix. Along the way we'll mutate `state` to become the new prefix.

{% highlight swift %}
let suffixes: UnfoldSequence<String, String> = sequence(state: seed) { (state: inout String) in
    guard let candidates = self.store[state] else { return nil }
    let nextSuffix = candidates.random
    state = state.droppingFirst() + " " + nextSuffix
    return nextSuffix
}
{% endhighlight %}

This could potentially go on forever, so we'll limit the output by only taking the first `maxWordCount` number of words. Then we'll reduce all these words into a single string joined by a space.

{% highlight swift %}
suffix.prefix(maxWordCount).joined(separator: " ")
{% endhighlight %}

The result will then be our `key` joined to all the words that grew out of it.

{% highlight swift %}
let result = seed + " " + suffixes.prefix(maxWordCount).joined(separator: " ")
{% endhighlight %}

Just like we didn't want to start the text in the middle of a sentence, we don't want it to end in the middle either. I'll truncate the string to that it doesn't extend past the last complete sentence using a custom extension on `String`.

{% highlight swift %}
let result = seed + " " + suffixes.prefix(maxWordCount).joined(separator: " ").truncatingToCompleteSentence()
{% endhighlight %}

Alright. That should get us off the ground. Now we can test it out.

## Running with the Text

First off, we need some text. And what better source text for fake text than [a repository of transcripts] of speeches made by Donald Trump. If you're following along in a playground you can copy the text files to the "resources" directory and they will be accessible to your code.

[a repository of transcripts]: (https://github.com/proxpero/trump-talk)

{% highlight swift %}
import Foundation
let files = [
    "comments-about-women-oct-2016"
    "trump-inauguration-speech-jan-2017",
    "cia-jan-2017",
    "black-history-month-2017",
    "prayer-breakfast-feb-2017"
]
func text(for file: String) -> String {
    let url = Bundle.main.url(forResource: file, withExtension: "txt")!
    return try! String(contentsOf: url)
}
let input = files.map { text(for: $0) }.joined(separator: "\n")
let chain = Chain(text: input, prefixLength: 2)
for _ in 1...20 {
    let text = chain.generateText(maxWordCount: 50)
    print("\(text)\n")
}

{% endhighlight %}

I get results like this.

> Well this is Black History Month, so this is too bad, but we’ll go right through it. But the fact is, should have kept the oil. But okay. Maybe you’ll have another chance. But the fact is, should have kept the oil. I wasn’t a fan of Iraq.

> Trump an intellectual? Trust me, I’m like a magnet. Just kiss. I don’t know, that’s tough competition. Which way? Oh, you’re finished?

> During the campaign, I'd go around with Ben to a lot better wages. We're gonna work together. This is your country. What truly matters is not a bad thing. He's respected all over the seven. But I met Mike Pompeo, and it was very interesting that Donald  Jr., whose incredible example is unique in American history.

> General Flynn is right over here. Put up your hand. What a good guy. And Reince and his whole group is going to straighten it out. Believe me. When you hear about the numbers. We did a fantastic job. And we’re going to bring those dreams back.

> I travel the country are five words that never, ever stop asking God for the wonderful introduction. So true, so true. I said it yesterday it has to know him anyway. But you’re going to be getting a total allegiance to all Americans.

> I had a feud with the understanding that is the right of all nations to put their own notions first. At the center of this movement is a great honor. I also happen to like Churchill, Winston Churchill.

> God will always be protected. Most importantly, we will always give us so much backing. But you’re going to say, please don’t give us such incredible heroes and patriots. They are very, very beautiful. His family was there, incredible family, loved him so much, so devastated, but the ceremony was amazing.

Not too bad. We could improve it, but I'm going to leave it there for now. Thanks for reading!

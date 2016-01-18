---
layout: post
title: Visualizing Strings in the Swift Standard Library
comments: true
mathjax: false
ganalytics: true
---

### Apple Sample Code

I always get excited when Apple releases new [sample][mac] [code][ios]. And while I may not have the [inclination][sp] or the [patience][kext] to work through every project that comes out, I have learned much and more (and haven't yet touched the bottom), and not just the details of implementing new features. You can see the personalities of mothership engineers shining as much through their coding technique as through their comments. Of course, the products are often not more than one massive super-tricked-out control, ugly as sin. Who cares? Design may well entail a thousand *NO!*s, but the purpose here is precisely to exhaust a feature's capabilities. On the other hand, the code you see in there... How shall I say it? It is sometimes not beautiful. I get it. Let me not be counted among those who say that Apple engineers do not work hard. The *feature* is the thing itself, the sample code deserves a burner further back. Yet when the code **is** beautiful --- as many times it is --- there is an valuable lesson in that as well.

[mac]: https://developer.apple.com/library/prerelease/mac/navigation/#section=Resource%20Types&topic=Sample%20Code
[ios]: https://developer.apple.com/library/prerelease/ios/navigation/#section=Resource%20Types&topic=Sample%20Code
[sp]: https://developer.apple.com/library/prerelease/mac/samplecode/SignalProcessing/Introduction/Intro.html#//apple_ref/doc/uid/TP40016163
[kext]: https://developer.apple.com/library/prerelease/mac/samplecode/enetlognke/Introduction/Intro.html#//apple_ref/doc/uid/DTS10003579

### The Swift Standard Library Playground

<div class="message">
<blockquote>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
And every phrase<br/>
And sentence that is right (where every word is at home,<br/>
Taking its place to support the others,<br/>
The word neither diffident nor ostentatious,<br/>
An easy commerce of the old and the new,<br/>
The complete consort dancing together)<br/>
Every phrase and every sentence is an end an a beginning,<br/>
Every poem an epitaph.<br/>
</blockquote>
T. S. Eliot, <i>from</i> <strong><a href="http://www.coldbacon.com/poems/fq.html">The Four Quartets</a></strong> 
</div>

I like Swift. I think of Swift as the proper language for a slim block of glass and space-grey aluminium. Swift is chamfered. Apple has an interest in polishing its public Swift code. 
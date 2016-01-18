---
layout: post
title: A Simple Drag-and-Drop View in Swift
comments: false
mathjax: true
ganalytics: true
---

## The Problem

I had a view with a simple vertical stack of subviews and I wanted to give the user the ability to reorder the subviews using drag-and-drop. Instead of implementing AppKit's drag-and-drop API, I wrote my own mechanism by hooking into `NSResponder`'s `mouseDragged(...)` method and `NSWindow`'s `trackEventsMatchingMask(...) -> Void)`. That way I could control the logic at the relevant steps during the user's action, without relying on AppKit's unseen magic. Finally, I refactored the logic into protocol extensions.





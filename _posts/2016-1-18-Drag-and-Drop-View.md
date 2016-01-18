---
layout: post
title: A Simple Drag-and-Drop View in Swift
comments: false
mathjax: true
ganalytics: true
---

## The Problem

I had a view with a simple vertical stack of subviews and I wanted to give the user the ability to reorder the subviews using drag-and-drop. Instead of implementing AppKit's drag-and-drop API, I wrote my own mechanism by hooking into `NSResponder`'s `mouseDragged(theEvent: NSEvent)` method and `NSWindow`'s `trackEventsMatchingMask(mask: NSEventMask, timeout: NSTimeInterval, mode: String, handler trackingHandler: (NSEvent, UnsafeMutablePointer<ObjCBool>) -> Void)`. That way I could control the logic at the relevant steps during the user's action, without relying on magic I couldn't see.





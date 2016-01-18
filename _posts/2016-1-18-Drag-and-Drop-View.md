---
layout: post
title: A Simple Drag-and-Drop View in Swift
comments: false
mathjax: false
ganalytics: true
---

## The Problem

I had a view with a simple vertical stack of subviews and I wanted to give the user the ability to reorder the subviews using drag-and-drop. Instead of implementing AppKit's drag-and-drop API, I wrote my own mechanism by hooking into `NSResponder`'s `mouseDragged(...)` method and `NSWindow`'s `trackEventsMatchingMask(...) -> Void`. That way I could control the logic at the relevant steps during the user's action, without relying on AppKit's unseen magic. Finally, I refactored the logic into protocol extensions.

## The Setup

I start with a view that contains the subviews I need to reorder. Following Cocoa's drag-and-drop API, I call this container view the Target. There are two important methods to implement. The first is `layoutSubviews(subviews: [NSView])`. 

{% highlight swift %}
    func layoutSubviews(subviews: [NSView]) {
        
        removeConstraints(constraints)
        var prev: NSView?
        
        for subview in subviews {
            
            addConstraints(NSLayoutConstraint.constraintsWithVisualFormat("H:|[subview]|", options: [], metrics: nil, views: ["subview": subview]))
            addConstraint(
                NSLayoutConstraint(
                    item:       subview,
                    attribute:  .Top,
                    relatedBy:  .Equal,
                    toItem:     prev != nil ? prev!     : self,
                    attribute:  prev != nil ? .Bottom   : .Top,
                    multiplier: 1,
                    constant:   prev != nil ? 1         : 0)
            )
            prev = subview
        }
        
        if prev != nil {
            addConstraint(NSLayoutConstraint(item: prev!, attribute: .Bottom, relatedBy: .Equal, toItem: self, attribute: .Bottom, multiplier: 1, constant: 0))
        }
    }
{% endhighlight %}



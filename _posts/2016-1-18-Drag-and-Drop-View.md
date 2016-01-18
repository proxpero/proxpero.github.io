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
        if subviews.isEmpty { return }
        
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
        
        addConstraint(NSLayoutConstraint(item: prev!, attribute: .Bottom, relatedBy: .Equal, toItem: self, attribute: .Bottom, multiplier: 1, constant: 0))
        
    }
    
{% endhighlight %}

You pass into this method an array of subviews and it will use autolayout to arrange the subviews from top to botton to match the order of the array. It assumes these subviews have already been added to the view: the superview never adds or removes any of these subviews during the drag operation.

## The Method

Before looking at the second method the Target view is required to implement, I'll explain the basic theory behind the solution. When the user tries to drag a view (I call it the "Source View", following Cocoa's drag and drop documentation), the `mouseDragged(theEvent: NSEvent)` method is called on the view (because it inherits from `NSResponder`). My Source view subclass overrides this method in order to tell its superview that a drag is beginning. It does this by calling that second method on the superview I referred to earlier. 

{% highlight swift %}

	// In the subview, an NSView subclass

    override func mouseDragged(theEvent: NSEvent) {

        if let target = superview as? MyTargetView {
            target.reorderSubview(self, withEvent: theEvent)
        }
        
    }

{% endhighlight %}

The superview, what I'm calling the Target, now knows to do several things. Here's the full method, I'll explain the steps afterward.

{% highlight swift %}

	// In the superview, an NSView subclass

    func reorderSubview(subview: Source, withEvent event: NSEvent) {
        
        let position        = NSMaxY(frame) - NSMaxY(subview.frame)
        let initial         = convertPoint(event.locationInWindow, fromView: nil).y
        
        let draggingView    = subview.draggingView
        
        let draggingConstraints = [
            NSLayoutConstraint(item: draggingView, attribute: .Top,      relatedBy: .Equal, toItem: self, attribute: .Top,       multiplier: 1, constant: position),
            NSLayoutConstraint(item: draggingView, attribute: .Leading,  relatedBy: .Equal, toItem: self, attribute: .Leading,   multiplier: 1, constant: 0),
            NSLayoutConstraint(item: draggingView, attribute: .Trailing, relatedBy: .Equal, toItem: self, attribute: .Trailing,  multiplier: 1, constant: 0)
        ]
        
        addSubview(draggingView)
        addConstraints(draggingConstraints)
        subview.hidden = true
        
        var previous = initial
        
        window?.trackEventsMatchingMask([.LeftMouseUpMask, .LeftMouseDraggedMask], timeout: NSDate.distantFuture().timeIntervalSinceNow, mode: NSEventTrackingRunLoopMode) { [unowned self] (dragEvent, stop) in
            
            guard dragEvent.type != NSEventType.LeftMouseUp else {
                
                draggingView.removeFromSuperview()
                self.removeConstraints(draggingConstraints)
                subview.hidden = false
                stop.memory = true
                return
                
            }
            
            let next = self.convertPoint(dragEvent.locationInWindow, fromView: nil).y
            
            draggingConstraints[0].constant = position + initial - next
        
            let middle      = NSMidY(draggingView.frame)
            let top         = NSMaxY(subview.frame)
            let bottom      = NSMinY(subview.frame)
            
            let movingUp    = next > previous && (middle > top)
            let movingDown  = next < previous && (middle < bottom)
            
            func moveSubview(direction: DragDirection) {
                
                let index = self.subviews.indexOf(subview)!
                let removeIndex: Int
                let insertIndex: Int
                
                switch direction {
                    case .Up:
                        removeIndex = index
                        insertIndex = index - 1
                    case .Down:
                        removeIndex = index + 1
                        insertIndex = index
                }
                
                let temp = self.subviews.removeAtIndex(removeIndex)
                self.subviews.insert(temp, atIndex: insertIndex)
                self.layoutSubviews(self.subviews.filter { $0 != draggingView })
                self.addConstraints(draggingConstraints)
                
            }
            
            if movingUp && subview != self.subviews.first! {
                moveSubview(.Up)
            }
            
            if movingDown && subview != self.subviews.last! {
                moveSubview(.Down)
            }
            
            previous = next
            
        }

{% endhighlight %}


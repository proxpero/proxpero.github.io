---
title: Easy UI Storyboard Extensions
summary: I create an extension on <code>UIStoryboard</code> to easily access Main.storyboard and instantiate custom <code>UIViewController</code> subclasses by type without littering the codebase with hardcoded strings.
tags: [Swift,UIStoryboard]
---

If you use storyboards, chances are that sooner or later you'll need to access them in code, for example to instantiate a view controller. In that case you can use the UIStoryboard initializer to create the storyboard you by providing its name.

{% highlight swift %}
let main = UIStoryboard(name: "Main", bundle: nil)
{% endhighlight %}

Xcode will look for a storyboard with that name in the main bundle. But its smells to hardcode strings. I'd rather use a static property.

{% highlight swift %}
let main = UIStoryboard.main
{% endhighlight %}

{% include toc.html %}

Then I can take advantage of code completion. The "Main" storyboard is common enough that it might as well be incorporated into an extension on `UIStoryboard`.

{% highlight swift %}
extension UIStoryboard {
  public static var main: UIStoryboard {
    return UIStoryboard(name: "Main", bundle: nil)
  }  
}
{% endhighlight %}

It's a rare case however that I'll need to create the main storyboard. More often I'll want to instantiate a view controller from a storyboard using its identifier, and force cast it to the corresponding subclass.

{% highlight swift %}
let vc = storyboard.instantiateViewController(withIdentifier: "MyViewController") as! MyViewController
{% endhighlight %}

When I follow the pattern in this example and use the name of the subclass as the identifier, I can use this extension:


{% highlight swift %}
extension UIStoryboard {
  public func instantiate<A: UIViewController>(_ type: A.Type) -> A {
      guard let vc = self.instantiateViewController(withIdentifier: String(describing: type.self)) as? A else {
          fatalError("Could not instantiate view controller \(A.self)") }
      return vc
  }
}
{% endhighlight %}

Now I can get the subclass without casting and with the benefit of autocompletion instead of typing out the string.

{% highlight swift %}
let vc = storyboard.instantiate(MyViewController.self) // Returns a non-optional instance of `MyViewContoller`
{% endhighlight %}

Not bad.

> Q: What if the view controllers are identified according to rules beyond my control, with the result that I can't rely on an identifier being identical to its view controller's class name?

In the case where you need to match the names in the storyboard file with the names you use in code, it's best to limit this mapping to a single location in your project. Make so you only have to type the string correctly once, and every other time you'll get confirmation of correctness from autocompletion and the compiler.

Let's start by modifying the extension above to take an optional identifier:

{% highlight swift %}
extension UIStoryboard {
    public func instantiate<A: UIViewController>(_ type: A.Type, withIdentifier identifier: String? = nil) -> A {
        let id = identifier ?? String(describing: type.self)
        guard let vc = self.instantiateViewController(withIdentifier: id) as? A else {
            fatalError("Could not instantiate view controller \(A.self)") }
        return vc
    }
}
{% endhighlight %}

Because this version includes a default value for `identifier`, it will produce a second type signature identical our first version's signature, so that it can replace that version without having to modify any callers elsewhere in your codebase. Now, you can provide your own identifier, but if `identifier` is `nil`, the name of the type of the view controller will be used instead.

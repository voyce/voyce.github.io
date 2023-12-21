---
date: 2010-10-01 17:00:53+00:00
description: ''
excerpt: You can no longer create public static fields in F# 2.0. Why was the change
  made and what impact does it have on WPF development?
featuredImage: ''
slug: /public-static-fields-gone-from-f-2-0
template: blog-post
title: Public static fields gone from F# 2.0
categories:
- .NET
- F#
- Software Development
- WPF
tags:
- DependencyProperty
- F#
- IL
- wpf
---

There have been quite a few changes in F# version 2.0, which shipped as the first "official" version of the language as part of Visual Studio 2010. Most of the changes are detailed in various release notes on Don Syme's blog and other places, but unfortunately one of the more significant changes passed me by, and turned out to be quite significant in the context of WPF development: public static fields are no longer supported. But what does this mean? 


### The change


The change itself is simple: static fields can no longer be public. Static fields can still be created, but they must be private. 

In pre-2.0 versions of F# it was possible to declare a static field on a type like this:

    
    
    type MyType() =
        [<defaultvalue>]
        static val mutable public MyProperty : int
    


Resulting in a type containing a static field, as you'd expect:

    
    
        .field public static int32 MyProperty
    


The code now generates the compiler error:

`error FS0881: Static 'val' fields in types must be mutable, private and marked with the '[<DefaultValue>]' attribute. They are initialized to the 'null' or 'zero' value for their type. Consider also using a 'static let mutable' binding in a class type.`

Notice that there are already some gnarly aspects to the definition of the property; notably the use of the `[<DefaultValue>]` attribute, which indicates that the field is un-initialized. This gives us a hint that there might be some inherent problems.


### Why do we even need them?


In a word or three: WPF dependency properties. 

The recommended way of implementing dependency properties in other .NET languages is to use public static fields, e.g. in C#:

    
    
        public static readonly DependencyProperty MyPropertyProperty =
            DependencyProperty.Register( 
              "MyProperty", typeof(int), typeof(MyType)); 
    


This is one of the few places where C# is terser than F#. Here we can declare and define the value in one line (ish), whereas F# requires a separate declaration of the field (as above) then initialisation in the static constructor (`static do`).


### Why was it removed?


I got the answer from the horse's mouth. Don Syme said that:


<blockquote>
We deliberately removed the ability to create public static fields in Beta2/RC, because of issues associated with initialization of the fields (i.e. ensuring the “static do” bindings are run correctly, and if they are run, then they are run for the whole file, in the right order, with no deadlocks or uninitialized access when initialization occurs from multiple threads).</blockquote>


You can imagine how this would be a problem; there would need to be a way of ensuring that whichever static field was accessed it caused the static constructor to run, which may itself access static fields. All pretty nasty. In fact, Don mentioned that C# suffers from much the same synchronisation issues, but just tends to be used in a way that means it's less likely to be noticed!



### The alternative


At least in theory it's possible to create a type that uses static properties rather than static fields to store its registered DependencyProperty information.

    
    
    type Foo () =
        inherit FrameworkElement()
        static mutable private _myPropertyInternal : DependencyProperty 
        static member this.MyProperty with get () = _myPropertyInternal
    


Whether or not this works depends very much on how calling code accesses DependencyProperty information. If it uses reflection to access the field directly it will obviously fail. But if it uses a more robust/flexible method then it should be OK. Empirically it seems that the WPF framework code itself does the latter, for instance, when it's instantiating objects from XAML, and it works properly independently of how it's implemented. 


### The conclusion


So the conclusion is "wontfix": the behaviour is by design. Unfortunately it has the effect that it's no longer possible to create a type with an identical IL "signature" in both F# and C#. It seems a bit of a shame, but I guess the trade-off is that we're protected against insiduous initialisation issues, so it will probably turn out to be the right thing in the long term.

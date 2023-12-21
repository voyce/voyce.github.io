---
date: 2009-12-14 11:05:02+00:00
description: ''
excerpt: What F# language and syntax features are required to implement a fundamentally
  object-oriented WPF custom control?
featuredImage: ''
slug: /a-wpf-custom-control-in-fsharp
template: blog-post
title: A WPF custom control in F#
categories:
- .NET
- F#
- Software Development
- Windows
- WPF
tags:
- .NET
- F#
- guidattribute fsharp
- wpf
---

In the world of WPF with its powerful templating support, you're much less likely to need to build a custom control from scratch than you are with legacy Windows GUI frameworks. For the vast majority of scenarios it's possible to take an existing control and modify its appearance and behaviour to get what you need. However it is still possible and sometimes necessary to build something in code. The other day I was looking at creating one - using F# of course - and realised that a skeleton control serves as a good example of the kind of cross-paradigm features the language offers. They're the kind of things that make it possible to use functional F# with inherently imperative .NET languages and frameworks like WPF.
<!-- more -->
Let's start by looking at the code for the control in its entirety, and then we'll break it down bit-by-bit:

    
    
    type public MyControl() =
        inherit ItemsControl()
    
        [<defaultvalue>]
        static val mutable FooProperty : DependencyProperty
    
        static member OnFooChanged (dob:DependencyObject) args =
            (dob :?> MyControl).Update ()
    
        [<system.componentmodel.bindable(true)>]
        member public this.Foo
            with get() : string  = string (base.GetValue(MyControl.FooProperty))
            and  set(r : string) = base.SetValue(MyControl.FooProperty, r)
    
        override this.OnPropertyChanged (args) =
            match args.Property.Name with
            | "Foo" -> this.Update ()
            | _          -> ()
    
        member internal this.Update () =
            System.Diagnostics.Debug.WriteLine (sprintf "Updating %A" (base.GetValue(MyControl.FooProperty)))
    
        static do
            let metadata = PropertyMetadata(null, PropertyChangedCallback (MyControl.OnFooChanged) )
            MyControl.FooProperty <- DependencyProperty.Register("Foo", typeof<string>, typeof<mycontrol>, metadata)
    




## Constructor


We don't have one! Well, that's not strictly true. There's no per-instance set-up that we need to do here, so instead we have a default, parameter-less constructor implied by the "empty brackets" syntax in the type declaration. If we wanted to execute some code in the constructor, we could add the following:

    
    
        do
            Debug.WriteLine "Constructing."
    


It's also possible to add further constructors (perhaps parameterised differently) but when using WPF bear in mind that instances of your class will often be created from a XAML declaration, which generally uses the default constructor and then sets properties as required. Mutable objects: ug.


## Inheritance


Our type derives from the WPF `ItemsControl` class using the `inherit` keyword. Of course, we're still subject to the single-inheritance limit of .NET (not a bad thing, if you ask me - no more tortured object hierarchies):

    
    
    type public MyControl() =
        inherit ItemsControl()
    


Note that we have to include the empty brackets on the inherited type name, as this will be constructed implicitly when our derived class is constructed. We can access the inherited class from elsewhere in the code using the `base` keyword.


## Static members


Dependency properties are a WPF construct that provide external storage of property values. They allow deep trees of objects to efficiently use lots of properties where they often have the default value, which is commonly the case in WPF. In order to use them with your class you have to do a few things, including creating a static value to hold the property and its metadata. We create a mutable static member for this:

    
    
        [<defaultvalue>]
        static val mutable FooProperty : DependencyProperty
    


Why a mutable static value? If you've used F# already you might be aware that it's also possible to declare an immutable static variable and its initial value in one shot with a `let`:

    
    
        static let FooProperty = DependencyProperty.RegisterProperty ("Foo", typeof<string>, typeof<mycontrol>)
    


Unfortunately, this results in your DP being private. Although the CLR property is still accessible, anything that attempts to access the DP directly - for instance, code within the WPF libraries - won't see it. This means you have to use the mutable style, which is unfortunate.


## Static methods


We've declared a static member function that is used to receive notifications when our DP is changed (although it's complete overkill in this example, because we've already declared our DP with metadata that tells it to notify us when it changes):

    
    
        static member OnFooChanged (dob:DependencyObject) args =
            (dob :?> MyControl).Update ()
    


As you'd expect, there's no `this` parameter on the signature, a static member doesn't have any implicit object instance to work on. Luckily the arguments to most event functions include the DependencyObject that raised the notification. That means we can downcast dynamically to our expected type (with `: ?>`) and use it. Bear in mind that this is more like casting than `as` in C#, as it will throw an `InvalidCastException` at runtime rather than returning null.


## Properties


In order for our dependency property to be easily accessible we can expose it as a plain old CLR property. The implementation of the getter and setter simply defers all of the actual work of storing and retrieving the value to the underlying dependency property.

    
    
        member public this.Foo
            with get() : string  = string (base.GetValue(MyControl.FooProperty))
            and  set(r : string) = base.SetValue(MyControl.FooProperty, r)
    


Notice how we have to cast the `obj` returned from `GetValue` into the correct type. This is another example of having to bridge the gap between the dynamically typed WPF property system and F#'s static typing.


## Overridden members


As well as providing new member functions and properties, we may need to override existing ones. Member functions marked `abstract` in the base class can be overridden using the `override` keyword:

    
    
        override this.OnPropertyChanged (args) =
            match args.Property.Name with
            | "Foo" -> this.Update ()
            | _          -> ()
    


The F# compiler provides the same kind of checking that the C# compiler does, warning you if you inadvertantly hide a base class function by creating an override with the same name but not marking it with `override`.


## Static constructor


As mentioned before, we use the static constructor to initialise our mutable static variables. The syntax is similar to the `do` syntax of a normal constructor, with the addition of the `static` keyword:

    
    
        static do
            let metadata = PropertyMetadata(null, PropertyChangedCallback (MyControl.OnFooChanged) )
            ...
    


Static constructors are run once per class, regardless of how many instances of the class you have. WPF relies quite heavily on static, class-based functionality; mostly because a lot of what's set-up is per-class configuration - it's not going to change during the lifetime of the application - so you may find yourself doing a fair amount of work in a static constructor. 

So, there's a quick run around some of the object-oriented features of the F# language: classes, inheritance, instance constructors, overridden member functions, static member functions and constructors. You can see how using WPF means you lose some of the benefits of the F# language; notably immutability and static typing. If you're an experienced functional programmer getting deep into creating WPF or Silverlight custom controls you may find yourself using these OO constructs more than you'd like. Although F# makes it relatively painless in practice, mixing this heavily object oriented style of programming with a functional approach can still be a little hard to stomach at times.

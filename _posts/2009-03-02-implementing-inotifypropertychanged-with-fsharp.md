---
date: 2009-03-02 12:15:29+00:00
description: ''
featuredImage: ''
slug: /implementing-inotifypropertychanged-with-fsharp
template: blog-post
title: Implementing INotifyPropertyChanged with F#
categories:
- F#
tags:
- event
- F#
- INotifyPropertyChanged
---

I like F# for a lot of things, but, man, is it a pain to support events. In C# it's trivial to implement an interface like INotifyPropertyChanged consisting only of an event, but in F# you have to jump through some hoops to map native functions to delegates/events. F# is generally much terser than C# and other .NET languages, but not in this case. After spending some time the other day trying to figure out the right combination of syntax and helper functions (and unsucessfully googling for it), I thought I'd upload a bare-bones implementation here as an aide-memoire.

    
    
    open System.ComponentModel
    
    type MyObject() =
        let mutable propval = 0.0
    
        let event = Event<_, _>()
    
        interface INotifyPropertyChanged with
            member this.add_PropertyChanged(e) =
                event.Publish.AddHandler(e)
            member this.remove_PropertyChanged(e) =
                event.Publish.RemoveHandler(e)
    
        member this.MyProperty
            with get() = propval
            and  set(v) =
                propval <- v
                event.Trigger(this, new PropertyChangedEventArgs("MyProperty"))
    


It turns out that in F# version 1.9.6.16 there's a slightly more concise syntax for this, as pointed out by Rei in the comments (thanks!). It uses the CLIEvent attribute to hook up the .NET event:

    
    
    open System.ComponentModel
    
    type MyObject() =
        let mutable propval = 0.0
    
        let propertyChanged = Event<_, _>()
        interface INotifyPropertyChanged with
            [<clievent>]
            member x.PropertyChanged = propertyChanged.Publish
    
        member this.MyProperty
            with get() = propval
            and  set(v) =
                propval <- v
                propertyChanged.Trigger(this, new PropertyChangedEventArgs("MyProperty"))
    

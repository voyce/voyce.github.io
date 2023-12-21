---
date: 2010-11-29 23:24:46+00:00
description: ''
excerpt: Discriminated unions for the rest of us.
featuredImage: build/gatsby/www.ianvoyce.com/assets/2010-11-29-fsharp-positive-discrimination_pattern_warning-300x71.png
slug: /fsharp-positive-discrimination
template: blog-post
title: 'Beginning F#: Positive Discrimination'
categories:
- .NET
- F#
- Finance
tags:
- F#
- functional programming
---

(or "Discriminated unions for dummies like me", or "Tagged unions for the rest of us")

Discriminated unions are one of those things in the lexicon of functional programming that can often sound baffling to "outsiders"; it's almost up there with monads and currying. But in practice they're simple and incredibly useful. I thought I'd try and show a concrete example of where they can be used in a way which is more powerful and robust than the equivalent OO approach.
<!-- more -->


### The usual suspects


When people first start looking at functional programming, they often come across examples that implement a compiler, and this is their first exposure to discriminated unions. Unfortunately these examples tend to use complex recursive discriminated unions to represent the language syntax tree, which can be a bit of turn-off for anyone who's not a language geek.



### A problem


The way I like to think of DUs is a means of easily and precisely specifying a set of types that we're expecting to work with. 

What do I mean by this? Well, imagine you're writing some code that can work on various different types, and you need to identify some common aspect to identify them. For example, you're valuing different financial product types: swaps, swaptions, caps and floors. Although each of these trade types would be represented differently, you may want to perform the same operation across all of them.



### The OO approach


The normal OO approach would be to use an interface; implemented by all of the types and consumed by our method. For instance:

    
    
    class Swap : public IValuable { ... }
    class Swaption : public IValuable { ... }
    ...
    public double ValueTrades(IValuable[] trades) { ... }
    


But this isn't particularly elegant or easy. For a start it requires that the type implement the interface - what if you're working with externally provided types that can't be changed? The other thing is that interfaces tend to give you a false sense of security about type safety. Given that we know there's an object implementing the interface, I've all too often seen OO code taking an interface parameter then immediately type-casting it to another interface that the underlying object is expected to support. Run-time typing failures FTL.     



### The union


So how could we implement this using discriminated unions instead?

    
    
    type Trade = 
        | SwapTrade of Swap
        | SwaptionTrade of Swaption
    


What we've done is create a new type that consists of exactly one of its possible cases. We identify each case with a tag (in fact, these types are often called tagged unions) called SwapTrade and SwaptionTrade and separate them with the bar '|'. 

Easy, isn't it? But, err, how do we get to the actual thing we want? Well that's where another functional programming staple steps in: pattern matching. We can add some code that will match against the possible union cases and allow us to get at the type itself:

    
    
    let valueTrades (trades : Trade list) = ...
        trades |> List.iter (fun t ->
            match t with 
            | SwapTrade s -> (* do something with s *) () 
            | SwaptionTrade st -> (* do something with st, which is of type Swaption *) ()
            )
    


[caption id="attachment_1082" align="alignright" width="300" caption="Pattern matching warning"][![Pattern matching warning](http://www.ianvoyce.com/wp-content/uploads/2010/10/pattern_warning-300x71.png)](http://www.ianvoyce.com/wp-content/uploads/2010/10/pattern_warning.png)[/caption]And one of the big advantages here is that we get compile time checking; the compiler will warn us if we fail to deal with any of the union cases. In other words, if we add a new case to the `Trade` union, we'll get a warning if we fail to deal with it wherever we've pattern-matched against it. In fact, that's the fundamental difference between this and the interface approach; in theory the interface approach means we can chuck anything we like at it; as long as it supports the interface it will work. In the discriminated union version we're being explicit about exactly what types we can handle.

Hope you found this useful. As a long-term OO developer moving to functional programming it certainly helps me to see how aspects of FP can be applied to common OO problems.

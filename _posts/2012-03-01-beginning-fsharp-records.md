---
date: 2012-03-01 17:53:08+00:00
description: ''
excerpt: An introductory look at record types, a useful F# language feature.
featuredImage: ''
slug: /beginning-fsharp-records
template: blog-post
title: 'Beginning F#: Records'
categories:
- F#
- Software Development
tags:
- .NET
- c++
- F#
- guidattribute fsharp
- introduction
- types
---

In the second of an unknown number of parts in my series of [Beginning F#](http://www.ianvoyce.com/index.php/2010/11/29/fsharp-positive-discrimination/) posts, I'll be talking about record types. They're a useful and powerful F# feature that you'll probably find yourself using very widely. I'll take a look at what they are, how they're used and how they integrate with the rest of the language. 
<!-- more -->


### What are they?


In technical terms, they're an algebraic data type. I won't attempt to precisely define that myself, because [this page](http://merrigrove.blogspot.com/2011/12/another-introduction-to-algebraic-data.html) does a much better job than I ever could. Suffice it to say that a record type is a combination (product) of named values of other types. 

For example we can take an int and a string and construct a record type like this:

    
    
    type MyRecord =
        {
        foo : int
        bar : string
        }
    


Of course, you can then use these record types as fields in other records:

    
    
    type AnotherRecord =
        {
        r : MyRecord
        x : string
        }
    


It might look like this is similar to using a C# `struct`, but there are some significant differences. For instance, record types can't be generic like structs can, and they're .NET reference types, not value types, although they don't necessarily _act_ like reference types, we'll talk about that later.


### Benefits


So why might you want to use records? 




  * They help to encourage strong typing. You can use tuples in some of the situations where you'd use records, but in my opinion the ability to name fields gives records the edge when it comes to code readability.


  * They encourage immutability. Although (some would say unfortunately) they don't require it, see below.


  * They integrate well with other language features such as pattern matching


  * They're concise

   


### Defining records


The syntax for defining new record types is pretty straightforward. 

    
    
    type MyRecord =
        {
        foo : int
        bar : string
        }
    


This is similar to F# [class definition syntax](http://msdn.microsoft.com/en-us/library/dd233205.aspx), but notice there are no implicit (`type MyRecord () =...`) or explicit constructors. Records have to be constructed in a specific way using record expressions that specify all field values:  

    
    
    let r = { foo = 1; bar = "a" }
    


Note that, as you'd expect, the F# compiler will attempt to infer the type, you don't have to declare the type of r. If there's ambiguity in the current scope, with multiple types having the same fields you can specify the type explicitly on either side, or fully specify one or all of the fields.

    
    
    // Specify type of record
    let r2 = { foo = 1; bar = "a" } : MyRecord
    // Specify type of function
    let r3 : MyRecord = { foo = 1; bar = "a" } 
    // Specify field
    let r4 = { MyRecord.foo = 1; bar = "a" } 
    


Another extremely useful feature of record types is that as well as fields, they can also include member functions, static functions, and even implement interfaces:  

    
    
    type Vector =
        {
        X : float
        Y : float
        Z : float
        }
        /// Member function has access to fields 
        member this.add() = this.X + this.Y + this.Z
        /// Static member, here is passed an instance as a parameter 
        static member foo2 v = v.X + v.Y + v.Z
        /// Implement an interface with members
        interface IDisposable with 
            member this.Dispose() = ()
    


At this point they're starting to look an awful lot like good old-fashioned OO classes. The big difference is really in how they're constructed; it's done in one-shot via record expressions, rather than with a multi-step instance create and modify. Because of this you can't provide any code that runs at construction time, as you would with `do` statements in F# classes.


### Modifying records


One of the surprising things - that I only discovered recently - is that you can actually modify record fields post-construction, provided they're explicitly declared as mutable:

    
    
    type Person =
        {
        Name : string
        mutable Age : int
        }
    

 
I consider this pretty nasty, and I have to say that I've never actually used records in this way. It's typical of the way in which F# uses functional programming idioms like immutability in a pragmatic way, encouraging good practice but not strictly requiring it.

Personally I prefer to pretend that records are immutable once they're constructed, and that modifying them requires copying. Luckily there's a concise way of creating copies with modified field values using `with`. Here's a function that takes an instance of a record type and returns a copy with a modified `foo` field:

    
    
    let newMyRecord r = { r with foo = r.foo + 1 }
    

 


### Pattern matching


A nice aspect of records is their ease of use in pattern matching. For instance you can very simply match against an individual field in the record, by using the underscore 'wildcard': 

    
    
    let hasZeroY =
        function
        | { X=_; Y=0.; Z=_ } -> true
        | _ -> false
    


Or you can even just specify an individual field; useful if you've got many fields:

    
    
     let xIsZero = function {X=0.} -> true | _ -> false
    




### Structural equality


By default, record types use structural equality. This means that although under the covers they're reference types, that's not how they're compared; the actual field values are used instead. This is an important difference between using simple C# classes and F# records. 

Because of the default reference equality semantics in C# (which essentially compares pointers), this code will output "False", despite the obvious similarity of `r1` and `r2`: 

    
    
    class CSharpRecord { public int a; }
    static void Main(string[] args)
    {
        var r1 = new CSharpRecord() { a=1 };
        var r2 = new CSharpRecord() { a=1 };
        Console.Write(r1 == r2);
    }
    



Whereas this returns 'true':

    
    
    type FSharpRecord = { a : int }
    let r1 = { a=1 }
    let r2 = { a=1 }
    r1 = r2
    



Apparently there's a performance bug in the currently released version of F# whereby records with lots and lots of fields (circa 300), will take a long time to compare. Personally, I'd argue that if you've got records with 300 fields, you've got bigger problems to solve, but you can at least work-around it by disabling structural equality and comparison with the `NoComparison` and `NoEquality` attributes:

    
    
    [<nocomparison;noequality>]
    type MyRecord = { X : float; Y : float; Z : float }
    


This removes the custom `Equals` and `CompareTo` implementation that would otherwise be generated by the compiler.


### How do they look?


If you do a bit of ILSpying (Reflector is _so_ last year), you'll probably be surprised at the amount of boilerplate that's been generated on your behalf by the compiler.

Most of it is the interfaces and overloads required for structural equality, but we also get 3 items for each field: the actual CLR field that provides the storage, a getter method and a property that calls the getter. It's more code than you'd have for the vanilla C# class we used above, but it's not particularly bad.

And to non-F# consumers, records look like perfectly normal types:

    
    
    var fr1 = new FSharpRecord(1);
    Console.Write(fr1.a);
    





### Summary


So there you go; a whistle-stop tour of records. If you want more info you can check out the [MSDN page](http://msdn.microsoft.com/en-us/library/dd233184.aspx), or, if you're feeling hardcore, the [language spec](http://research.microsoft.com/en-us/um/cambridge/projects/fsharp/manual/spec.html), but in the meantime... have fun!

---
date: 2009-04-24 21:19:17+00:00
description: ''
excerpt: A description of using F# language features and reflection to enable basic
  analysis of .NET IL (intermediate language).
featuredImage: ''
slug: /il-analysis-using-fsharp
template: blog-post
title: IL analysis using F#
categories:
- .NET
- F#
- Software Development
tags:
- .NET
- F#
- IL
---

I recently needed to determine which functions were called by some of our F# code. Naively, you can use existing tools like ildasm, to disassemble a .NET DLL and then search the resulting IL source code for references. The obvious problem here though, is that you're going to include _all_ references whether or not they're actually called. In some circumstances this isn't too bad, but in our case we pull in a great deal of shared library code, so you're likely to get lots of false positives.

There are some other options to more accurately determine whether the method you're interested in is actually called: run the code, or "almost" run it, by simulating the operation of the CLR. To radically understate; this is quite a lot of work. Yet another option is to statically analyse the original source code. This is generally easier than dynamic evaluation, but there are some serious and well known problems doing it exhaustively, that can result in the complexity eventually converging with that of full dynamic analysis.

So broadly, we have 3 types of approches:<table >
<tr >
<td >_
Approach_


</td>
<td >_
Implementation_


</td>
<td >_
Accuracy_


</td></tr>
<tr >
<td >Disassembly
</td>
<td >Easy
</td>
<td > Superset
</td></tr>
<tr >
<td >Dynamic analysis
</td>
<td > Hard
</td>
<td > Exact
</td></tr>
<tr >
<td >Static analysis
</td>
<td > Medium
</td>
<td > Medium
</td></tr></table>
Anyone for a trade-off? Unsurprisingly I decided to look at implementing the third option. Although static analysis is normally performed on the source code itself, it's actually easier for us to use the generated IL, it certainly requires less gnarly parsing. We can also take some short cuts based on the fact that we're analysing F# code, more on that later.

We can use F#'s discriminated unions - a type that is constructed from one of many possible options - to describe the universe of IL instructions in a pretty concise way, e.g. (a partial example):

    
    
    type inst =
        | Nop
        | Break
        | Ldarg_0
        | Ldc_i4 of int32
        | Newobj of meth
    and field = FieldInfo
    and meth = MethodBase
    and typ = Type
    


This allows us to construct instances of `inst` by doing something like this in fsi (F# interactive):
`
> let i = Ldc_i4 2;;
val i : inst
`
You may have noticed that as well as the instructions that take simple types like int32, we also have ones that accept `meth`, which is an alias for System.Reflection.MethodBase, the base class for all methods, including constructors, which is what's used to construct a `Newobj`.

Now we have this discrimated union defined, we need a way to build instances of it. In the IL byte stream, instructions are stored as opcodes, an unsigned 16bit integer. Firstly we need to get the raw bytes representing the IL. Using Reflection, it's fairly easy given `m` of type `MethodInfo`:

    
    
        let body = m.GetMethodBody()
        let ilbytes = body.GetILAsByteArray()
        let ms = new IO.MemoryStream(ilbytes)
        ...
    


So now we have a stream of bytes, and we can use functions from System.IO to extract information in various sized pieces:

    
    
        let getByte _  = (byte (ms.ReadByte()))
        let i2 _ = readInt16 ms
        let i4 _ = readInt32 ms
        ...
    


As Harry Hill would say; "well, you get the idea with that". It's worth noting that these functions have a dummy argument (indicated by the
underscore). This is required because they have a side effect - reading from the stream, changing it's state - which is not obvious to the compiler, so if we omitted it the function would only be called once. Although adding the dummy arg is required, it does have the unfortunate consequence that we have to pass something (normally unit) which can look a little ugly in the normally terse F# world.

As the ECMA CIL spec describes, IL opcodes consist of either 1 or 2 bytes, in which case the first is always 0xFE. Now we can begin to implement something serious. Given `ms` of type `MemoryStream` we can write something that will convert it to instructions:

    
    
        match ms.ReadByte() with
        | 0xFE as lb ->
            // Two byte instruction, read further byte
            let hb = getByte()
            let i = ((uint16 lb) <<< 8 ) + (uint16 hb)
            let t =
                match i with
                | 0xfe01us -> Ceq
        | _ as b ->
            let t =
                match b with
                | 0x0 -> Nop
                | 0x1f -> Ldc_i4_s (getByte())
                | 0x20 -> Ldc_i4 (i4())
                | 0x73 -> Newobj (meth())
    


So we now have a function that will go from a method to a list of opcodes and operands (`MethodBase -> inst []`). These are essentially the same steps we would perform if we were writing an interpreter for a textual language; taking the source and transforming it into an abstract syntax tree. In that case it's a tree rather than a list, but the next step is pretty much the same anyway: we pattern match over it. This is the point where we can decide how we want to interpret the instruction stream.

    
    
            insts
            |> List.map (fun inst ->
                match inst with
                | Newobj(meth) ->
                    printf "NEW: %s.%s\n" meth.DeclaringType.Namespace meth.DeclaringType.Name
                | _ ->
                    ()
    


Here we need to make some compromises based on the problem domain. I'm not trying to create a general purpose static analyser, but one that will work on object code in a certain format - that generated by the F# compiler. As such we make some assumptions and use some knowledge about the internals of the compiler to get the result we're after. To be specific we're relying on the fact that the compiler generates types for closures, and we assume that closures will always be called, even though in reality they needn't be.

So based on this, we can put together something that, given an entry point - a particular method on a type - can recurse through the code, following references to other methods and types via the `Newobj`, `Call`, `Calli` and `Callvirt` instructions. This will build up a graph of all types referenced directly from our starting point. We also use our intimate knowledge of the purpose of F#'s `FastFunc` type (from which all functions are derived) and always follow its Invoke method if we find an instance of that type, even if it's not directly referenced.

There are some major caveats. Anything accessed purely via reflection will not be detected. And polymorphic objects passed in and accessed via interfaces will also be missed. Also, I don't attempt to do full flow analysis; e.g. following branch instructions etc, as this isn't a common pattern in fsc-generated IL.

Luckily in the particular cases I'm looking at, these shortcomings don't have a significant impact. Instead, we end up with a reasonably straight-forward and useful way of determining whether a particular function is called. It's already been used in anger to determine whether a buggy function was referenced from some release-candidate software.

As a little post-script: rather than writing your own library from the ground-up to do this, there are some "off-the-shelf" solutions that you can try. Notably the recently released [CCI](http://ccimetadata.codeplex.com/), a common compiler infrastructure out of Microsoft Research, that allows you to reverse engineer IL metadata. I haven't had a chance to have a good look at this yet, but it seems to do what we need for call graph analysis. There's also an API called AbstractIL - in the absil.dll assembly - that ships with and is used internally by the F# compiler toolset. This looks extremely powerful, but the API is complex and the documentation is poor. Depending on exactly what your motivation is for looking at this stuff, it's worth checking if these ready-made libraries will do what you need.

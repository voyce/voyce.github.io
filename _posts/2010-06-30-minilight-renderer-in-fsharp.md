---
date: 2010-06-30 17:19:43+00:00
description: ''
excerpt: Porting the Minilight global illumination renderer to F#.
featuredImage: build/gatsby/www.ianvoyce.com/assets/2010-06-30-minilight-renderer-in-fsharp_cornellbox-e.png
slug: /minilight-renderer-in-fsharp
template: blog-post
title: Minilight renderer in F#
categories:
- .NET
- F#
- Graphics
- Software Development
tags:
- .NET
- F#
- Graphics
- monte carlo
---

[caption id="attachment_961" align="alignleft" width="140" caption="Cornell Box in the evening"][![Cornell Box in the evening](build/gatsby/www.ianvoyce.com/assets/2010-06-30-minilight-renderer-in-fsharp_cornellbox-e.png)](build/gatsby/www.ianvoyce.com/assets/2010-06-30-minilight-renderer-in-fsharp_cornellbox-e.png)[/caption]I'm a sucker for eye-candy, and the other day I came across the beautifully lit renders produced by [Minilight](http://www.hxa.name/minilight/). It's a nice, minimal implementation of a global illumination renderer that's been ported to a wide variety of different languages from C to ActionScript. So of course, I couldn't resist trying to implement it in F#. 
<!-- more -->


### What is a "global illumination renderer"?


In short; a global illumination renderer attempts to simulate the way that light behaves in the real world. It bounces rays of light around the scene, noting which surfaces are hit, and how each of them affects the ray. Of course real light travels fast - at about the speed of light, in fact - and our simulated rays can't match that, so although the resulting effect looks natural, it's not a very efficient way of getting there. Graphics systems that need to be near real-time (i.e. games), use a variety of short-cuts to try and get the same effect without requiring the same amount of number crunching.

Even this implementation is forced to use a couple of techniques to improve performance, I won't go into them here, as they're explained pretty thoroughly on the Minilight site. 

Interestingly the algorithm uses a Monte Carlo technique to simulate the random path of the ray through the scene and the same technique is used in finance to model how the price of an asset will change through time. 



### Translating to F#


[caption id="attachment_970" align="alignright" width="150" caption="Room with sun and light, 800 paths per pixel"][![Room with sun and light, 800 paths per pixel](build/gatsby/www.ianvoyce.com/assets/2010-06-30-minilight-renderer-in-fsharp_roomfront.sun.ml-150x150.png)](http://www.ianvoyce.com/wp-content/uploads/2010/06/roomfront.sun.ml.png)[/caption]Luckily, the existing OCaml version is considered one of the reference implementations, and as ML is the spiritual predecessor of F# I didn't expect it to be too difficult to port. It wasn't. I'm no ML expert, so some of the language features were a bit of a surprise; things like the use of a dot on all of the operators, but these are pretty minimal syntactic differences.

Some of the ML constructs caused warnings from the F# compiler as I didn't use the explicit ML compatibility mode. For instance, the use of the `^` operator to concatenate strings:
`
warning FS0062: This construct is for ML compatibility. Consider using the '+' operator instead. This may require a type annotation to indicate it acts on strings.
`
Some other changes included:




  * Using dot rather than # notation to access class members


  * Using square bracket for array element access (`array.[n]` rather than `array.(n)`)


  * Removing `let...and` for non-recursive bindings



There was also the opportunity to make use of some of the .NET framework classes, things like `System.Drawing.Bitmap` to write pixels directly into a bitmap and then save it into a .PNG file. Much easier than trying to find something to read the .PPM file that the original version emits.    



### Potential improvements


One of the most obvious improvements that could be made is to make use of the parallelism support in F# and .NET. The algorithm itself is easily (if not embarrassingly) parallelisable, given that it iterates over the pixels in the output image, and performs independent operations on each of them. Unfortunately I haven't had a chance to look at this yet - and the Windows VM I'm using only has a single processor anyway, so on a purely selfish note, I wouldn't see any immediate performance improvements.  It should be as simple as changing the pixel loop to use `Parallel.For`, or creating an `async` computation expression.

Something else to note: I err'd on the side of caution when it comes to saving intermediate images, so every iteration it creates and writes the file. This will be doing lots of allocation and strictly unnecessary work, especially for simple scenes.

Given this approach is so completely compute bound it would be interesting to see how moving some of the calculations to a GPU would affect it. It might even be possible to use quotations for some of the inner loops and then generate Nvidia intermediate language (PTX) from them - that's something I've been wanting to do for a while. 

I did run the code through the Visual Studio 2010 profiler to get a rough idea of where the time was being spent; it looked like the majority was in the leaf of the calculations where numerical operations where being performed on the vectors, which is consistent with what we'd expect.



### Use the source


So, here's the source. No warranties implied, use at your own risk, your mileage may vary, all errors are mine etc, etc.
[minilight_fsharp](http://www.ianvoyce.com/wp-content/uploads/2010/06/minilight_fsharp.zip)

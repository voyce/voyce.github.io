---
date: 2012-02-28 22:23:20+00:00
description: ''
excerpt: There's a bug in 32-bit .NET 2.0 where assemblies are loaded twice, wasting
  valuable address space.
featuredImage: build/gatsby/www.ianvoyce.com/assets/2012-02-28-net-dlls-loaded-twice_vmmap_double_highlight-300x260.png
slug: /net-dlls-loaded-twice
template: blog-post
title: .NET DLLs Loaded Twice
categories:
- .NET
- Debugging
- Windows
tags:
- .NET
- address space
- dll
- VMMap
---

If, like me, you're still squeezing yourself into 32-bit Windows processes, you're probably, also like me, constantly keeping an eye on the virtual address space usage of your application. If you happen to have used something like vmmap to take a peek at your memory contents, maybe you've noticed something strange with some .NET assemblies: they're loaded twice! What's going on...?
<!-- more -->
Let's see how this looks in practice. 

I created a .NET 3.5 app that does nothing except load a library. The library (doublelib.dll) contains a class, so we can reference it from the app, and a large uncompressed 24-bit BMP to waste a bit of space. In real life you'll find lots and lots of code will also serve to waste space.[caption id="attachment_1517" align="alignright" width="300" caption="vmmap screenshot"][![vmmap screenshot](http://www.ianvoyce.com/wp-content/uploads/2012/02/vmmap_double_highlight-300x260.png)](http://www.ianvoyce.com/wp-content/uploads/2012/02/vmmap_double_highlight.png)[/caption]

If we ramp up [vmmap](http://live.sysinternals.com/vmmap.exe) and select the Images type, we can clearly see the 1.6mb DLL loaded twice. Click on the image for the full size screen shot.

You'll notice that this only affects DLLs that aren't in the GAC, and only affects .NET 2.0-based apps.

So let's cut to the chase: there's a hotfix available, and you can grab it from here: [http://support.microsoft.com/kb/981266](http://support.microsoft.com/kb/981266). It was reported a long time ago on MS Connect, but it seemed as if it had ended up in the `wontfix` pile... Then suddenly, a few months ago, a fix appeared. The only caveat is that it's an opt-in thing; I guess this, and the delay, is probably because 2.0 is a "legacy" - although still officially supported - version of the runtime. MS seem reluctant to mess with what's broadly working.

So, if you're squeezed for memory, give the hotfix a try and reclaim your address space...!



### Postscript


There's a terrible bit of advice/misinformation on that KB page about the hotfix. It suggests one of the fixes is to:


<blockquote>Run the application on a 64-bit operating system. 64-bit operating systems have a large virtual address space.</blockquote>


This might be true for a 64-bit process on a 64-bit OS (where it'll get a whopping 16TB), but in the case of a 32-bit app (which is specifically what the bug affects), you're only going to get a relatively small amount of additional address space by running on 64-bit Windows; in the realm of a few hundred extra MB out of a theoretical maximum (excluding use of special switches) of 2GB.

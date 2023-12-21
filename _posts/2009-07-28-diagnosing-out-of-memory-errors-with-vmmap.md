---
date: 2009-07-28 22:38:19+00:00
description: ''
excerpt: VMMap is a new tool from Mark Russinovich et al that's very useful for diagnosing
  virtual memory/address space exhaustion issues. I describe it here, and give some
  information that should help you interpret what it reports.
featuredImage: build/gatsby/www.ianvoyce.com/assets/2009-07-28-diagnosing-out-of-memory-errors-with-vmmap_overview-300x99.png
slug: /diagnosing-out-of-memory-errors-with-vmmap
template: blog-post
title: Diagnosing out of memory errors with VMMap
categories:
- Debugging
- Software Development
- WinDbg
- Windows
tags:
- memory
- private bytes
- VMMap
- win32
- WinDbg
---

The other day a colleague pointed me to a new tool from Mark Russinovich et al (ex-SysInternals) that turns out to be very useful for diagnosing virtual memory/address space exhaustion issues. I thought I'd describe it here, and give some - hopefully useful - extra information on what it reports.

(I had problems with WordPress choking on this long post, so I've split it into 2 parts. This the first part, the second part is [here](http://www.ianvoyce.com/index.php/2009/07/29/diagnosing-out-of-memory-errors-with-vmmap-part-2/)).

<!-- more -->
First things first: you can download it from [here](http://technet.microsoft.com/en-us/sysinternals/dd535533.aspx).



## Using VMMap


VMMap graphically displays the contents of your processes' virtual memory, which each type of memory colour coded. This means you can quickly get an idea of how all of your valuable 2GB address space is being used.

At this point it's worth pointing out that this article refers to the 32 bit Windows XP platform only, as that's what I use on a day-to-day basis. Moving to a 64 bit address space makes it so large that exhaustion of a process' address space is pretty damn unlikely, discounting your application doing something pathological.

[caption id="attachment_219" align="alignleft" width="300" caption="VMMap top pane"][![VMMap top pane](build/gatsby/www.ianvoyce.com/assets/2009-07-28-diagnosing-out-of-memory-errors-with-vmmap_overview-300x99.png)](http://72.47.193.211/wp-content/uploads/2009/07/overview.png)[/caption] Before digging in to the different types of allocation, you can quickly get a good overview using the top part of the window.  
The top bar, Virtual Memory Summary (5040K in this example), corresponds to the Committed memory column and the Working Set summary bar (1140K) below it corresponds to the Total WS column.

The number in the bottom left in the largest free block of address space. It's the same number as reported by `!address -summary` in WinDbg. See the post [here](http://www.ianvoyce.com/index.php/2009/08/21/largest-free-block-of-address-space/) if you want to calculate it programmatically. If this number is small (less than 64MB) you'll probably have out of memory issues, see below for more details.

**Note:** Generally the working set size is NOT the important number here; it's a common mistake that people make when looking at memory usage figures in something like Task Manager. The working set size is merely an indication of the amount of virtual memory that has recently been accessed and it's entirely possible to make the WS size number plummet by explicitly "flushing" the working set. You can do this by, for example, minimising a GUI application. As such, this is really not the number you want to be tracking. Instead, look at the Private Bytes or Virtual Memory Size counter in perfmon.

But back to VMMap. There are various different types of allocation reported. I'll describe some of the most interesting ones:



### Image


A DLL or other executable file, as typically loaded by `LoadLibraryEx`. You'll see the protection is Execute/Copy on Write; this is the standard for executables, where the OS will share the memory between processes unless they modify it - which is very rare.

You may also see the same file loaded as a mapped image. For instance, when using a COM server that contains an embedded type library resource, it will be loaded in both ways; once as an executable using `LoadLibrary` and once as a mapped file using `LoadTypeLib`. You may also notice that non-ngen'd .NET images are mapped **twice**. This appears to be a bug that Microsoft are aware of, there's a [bug](http://connect.microsoft.com/VisualStudio/feedback/ViewFeedback.aspx?FeedbackID=467560) logged in Connect for it.

You can expand the image entry to see the size of the image sections, the file header, code (.text), resources (.rsrc) and relocations (.reloc) for example.



### Mapped file


These are typically data files loaded using `CreateFileMapping`. Depending on how the mapping was created (whether an explicit maximum size was specified), either the whole of the file or some small section of it will be mapped. Of course, the whole point of memory mapped files is to be able to get access to sections of a file without having to load it all, so the chances are that only small files will be mapped in their entirety.

Memory mapped files are often used to share data between processes, so you may find that things put on the clipboard may appear as mapped address space, albeit transiently. This is especially obvious in Microsoft Office apps like Excel.



### Heaps


This is the meta-data associated with each heap in the process. The actual allocations from the heap are stored separately in the address space (more on that later, see Private Data).

Some people are surprised that there can be multiple heaps in a single process, but it's actually quite common practice in unmanaged code, especially where there are specific memory management requirements. MSXML is a widely used example of this; it does it's own garbage collection using it's own dedicated heaps.

There is always a default heap created by the loader during process creation, and others are created explicitly using `HeapCreate`. The address of the heap displayed by VMMap can be used with the WinDbg `!heap` command to delve more deeply into it's contents and structure. For instance, `!heap -m` will display it's segment details.



### Managed heap


Obviously this is only relevant for applications using .NET - managed apps themselves or unmanaged apps that implicitly load .NET assemblies in some other way, say, via COM. The number displayed is the sum of the generation 0, 1, 2 and large object heap sizes.



### Thread stack


Each thread in the OS has a stack that can grow to 1MB by default (although this is configurable at link time, and controllable programmatically). It starts as a block of uncommitted reserved memory, with the used space committed at the bottom, and a guard block that is used to determine when to expand the block downwards. VMMap helpfully displays the thread ID in the stack space row.

Remember that each thread will have it's own stack space, so spinning up threads is not free in terms of virtual memory use. See [here](http://msdn.microsoft.com/en-us/library/ms686774(VS.85).aspx) for more details on thread stack space.

Read on [here](http://www.ianvoyce.com/index.php/2009/07/29/diagnosing-out-of-memory-errors-with-vmmap-part-2/) for more...

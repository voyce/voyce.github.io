---
date: 2009-07-29 08:32:34+00:00
description: ''
featuredImage: build/gatsby/www.ianvoyce.com/assets/2009-07-29-diagnosing-out-of-memory-errors-with-vmmap-part-2_heap1.png
slug: /diagnosing-out-of-memory-errors-with-vmmap-part-2
template: blog-post
title: Diagnosing out of memory errors with VMMap - Part 2
categories:
- Debugging
- Windows
tags:
- address space
- heap
- perfmon
- private bytes
---

(I had problems with WordPress choking on this long post, so I've split it into 2 parts. The first part is [here](http://www.ianvoyce.com/index.php/2009/07/28/diagnosing-out-of-memory-errors-with-vmmap/). This is the second part).
<!-- more -->


### Private data


This is the data that is explicitly allocated by the process, or blocks of memory that contain the allocated data. So when you're allocating from the heap, for instance, using code like:

    
    
    char *p = new char[1024*1024];
    


you can see how the heap manager manages address space; creating reserved segments of a fixed size, then committing parts of them as required.
When a segment is full, it will create a new one of double the size.

Allocate first 256*1024 bytes:
[caption id="attachment_220" align="alignleft" width="293" caption="After allocating first 256KB"][![After allocating first 256KB](http://72.47.193.211/wp-content/uploads/2009/07/heap1.png)](http://72.47.193.211/wp-content/uploads/2009/07/heap1.png)[/caption]
  
Allocate another 256*1024 bytes:
[caption id="attachment_221" align="alignleft" width="293" caption="After allocating another 256KB"][![After allocating another 256KB](http://72.47.193.211/wp-content/uploads/2009/07/heap2.png)](http://72.47.193.211/wp-content/uploads/2009/07/heap2.png)[/caption]
  
Allocate yet another 256*1024 bytes:
[caption id="attachment_222" align="alignleft" width="293" caption="And another..."][![And another...](http://72.47.193.211/wp-content/uploads/2009/07/heap3.png)](http://72.47.193.211/wp-content/uploads/2009/07/heap3.png)[/caption]
  
Allocate another 256*1024 bytes. This time a new heap segment is created:
[caption id="attachment_223" align="alignleft" width="289" caption="And yet another..."][![And yet another...](http://72.47.193.211/wp-content/uploads/2009/07/heap4.png)](http://72.47.193.211/wp-content/uploads/2009/07/heap4.png)[/caption]  
Many heaps in the process, all attempting to create segments in this way can cause problems, see Heap contention.



## Causes of out of memory errors





### Fragmentation


If you're experiencing out of memory errors, the first thing to check is the largest free block size. Obviously if you're requesting more than the available size, your allocation will fail. Remember that large requests may be made implicitly by mechanisms that are outside your control. For instance, Windows heaps expand geometrically, attempting to grab a segment of double the previous size each time, e.g. 16, 32, 64, 128MB. As you can see, allocating a single byte when the 64MB segment is full will result in the heap manager trying to reserve a 128MB block.

So once you've seen that the largest free block is small, the question then is: why? It's either due to genuine memory exhaustion or address space fragmentation.



### Heap contention


Fragmentation can be due to multiple heaps contending for contiguous address space. This is typically the case when you have a mix of VC++ code that is built using different versions of the runtime libraries (msvcrt.dll, msvcr70.dll, msvcr80.dll, msvcr90.dll), each of which manages it's own heap.

Remember that heap segments need to be reserved as a contiguous block of address space, but if your virtual memory is fragmented, the heap manager may not be able to obtain one. In this case, it will fall-back to creating segments of smaller sizes. The problem is that there's a limit to the number of segments that can be created - a measly 32 in Windows XP - so if fragmentation causes it to create more, smaller segments this limit may be reached. If a new segment cannot be created you'll get out of memory errors.

If you suspect this, you can use `!heap -m` to see details of the segment count and sizes for each heap. To identity the heap associated with each version of the MSVC runtime, ensure you've got the appropriate symbols loaded and then use `dd msvcr80!_crtheap L1` to see the address.




### Address space exhaustion


It may also be possible that you _really have_ exhausted your 2GB address space. This can happen when your process wastes lots of address space due to allocation granularity. For example calling `VirtualAlloc` without an address will cause the OS to choose one for you, and as the documentation states, this will be rounded to a multiple of the allocation granularity, 64KB. So if you happen to allocate lots of objects of only a few bytes with direct calls to VirtualAlloc, you will waste almost 64KB a time. Although this might not seem significant in a 2GB address space, it all adds up.

One of the symptoms of address space exhaustion is DLLs failing to load. I noticed recently that a COM `CoCreateInstance` call was failing because the only address space left to load the DLL into was way up into the area usually reserved for OS DLLs such as ntdll.dll.




### Other tips


By default the allocations are ordered by Address (the first column) and, because things are generally allocated in increasingly higher locations in memory, this can serve as a useful "timeline" of the app's allocations. It's not guaranteed though: DLLs that have an explicit load address don't follow this pattern (for example, VBE6.DLL always loads at 0x65000000). You can use it to see roughly when threads and heaps are created and files are mapped though.



### Summary


So, I hope you find this information useful in interpreting the output from VMMap. It's a very good way of getting visibility on the state of your processes and it's certainly more intuitive than having to use the !address and !heap commands in WinDbg.

Good hunting!

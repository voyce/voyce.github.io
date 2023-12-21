---
date: 2009-09-03 22:51:35+00:00
description: ''
excerpt: How do you find the unmanaged COM object that's being referenced by a .NET
  object?
featuredImage: ''
slug: /getting-iunknown-from-__comobject
template: blog-post
title: Getting IUnknown from __ComObject
categories:
- .NET
- COM
- Debugging
- Software Development
- WinDbg
- Windows
tags:
- .NET
- COM
- mscorwks
- RCW
- WinDbg
---

I'm working in an environment with a lot of mixed managed (F#) and unmanaged (C++ COM) code. One of the big problems with this is the mix of lifetime management techniques; .NET uses garbage collection while COM relies on reference counting. Furthermore .NET garbage collection is somewhat non-deterministic, which adds further complexity.

So quite often in our mixed code-base, we find that the .NET garbage collection process doesn't kick in when we need it to. For instance, when we've allocated a lot of memory in the COM code that .NET isn't aware of. Memory exhaustion has to get pretty bad for the GC to occur at any other time than during a .NET allocation, either the system-wide low-memory event has to be signalled or an `OutOfMemoryException` needs to be thrown. In both of these cases it's probably too late to do anything about it.

In this case it's extremely useful to be able to see what .NET objects are still alive, and what COM objects they're hanging on to. Unfortunately this isn't as easy as it might seem.
<!-- more -->
The COM object itself hides within a weakly-typed `System.__ComObject` or a strongly-typed managed wrapper, depending on whether rich type information is available. Furthermore, a runtime controller RCW (runtime callable wrapper) is what actually holds a pointer to the object itself, and this structure is internal to mscorwks.dll.

So how can we untangle this and, on finding a `__ComObject` that happens to still be alive (i.e. is not reachable in the object graph and is therefore eligible for garbage collection) identify which COM object it's hanging on to.

First of all, let's see how many `__ComObjects` are still alive. In this case, it's only one (phew!):

    
    
    0:005> !DumpHeap -type __ComObject
     Address       MT     Size
    01453b74 79306e60       16
    total 1 objects
    Statistics:
          MT    Count    TotalSize Class Name
    79306e60        1           16 System.__ComObject
    Total 1 objects
    



And you remember the layout of .NET objects in memory, don't you? Of course you do! The 4 bytes prior to the address displayed (`01453b74`) are the "object header". The exact content of the header is apparently undocumented. Let's see what it contains (at least on a 32-bit platform, your mileage may vary):


    
    
    0:005> dd 01453b74-4 L1
    01453b70  08000002
    



According to various sources the object header contains 2 fields; a handle and a sync block index. If the object is an RCW, the handle is always 0x08000. You can use the index with SOS's `!syncblk` command to de-reference it:


    
    
    0:005> !syncblk 2
    Index SyncBlock MonitorHeld Recursion Owning Thread Info  SyncBlock Owner
        2 001e0fec            0         0 00000000     none    01453b74 System.__ComObject
    -----------------------------
    Total           3
    CCW             0
    RCW             1
    ComClassFactory 0
    Free            0
    



The sync block itself is an undocumented structure, but after a bit of investigation, it turns out that at offset 0x1c there is a pointer to a further structure that contains the "interop information":


    
    
    0:005> dd 001e0fec+1c L1
    001e1008  001e9510
    



And from this, we can obtain a pointer to the RCW itself. We're almost there!


    
    
    0:005> dd 001e9510+c L1
    001e951c  001e5380
    



The RCW is a pretty large structure, but for our purposes there are only a couple of interesting fields: the IUnknown pointer at 0x64, and the object's virtual function table pointer at 0x88. If you use `dds` you can easily see any symbols associated with these pointers:


    
    
    0:005> dds 01e5380+64 L1
    001e53e4  00ef6c24
    




    
    
    0:005> dds 01e5380+88 L1
    001e5408  00eb9710 rcwrepro!ATL::CComObject<ctestobject>::`vftable'
    



This is the salient information; we now know exactly what type of COM object we're dealing with. This is obviously a bit fragile, given that it relies on structures from mscorwks that may well change in newer versions of the runtime (I'll check on .NET 4 when I get a chance). It's also a bit of a pain to go through all these steps manually in WinDbg, so I put together a simple extension DLL to do it automatically given the address of the __ComObject. I'll upload that and blog about it soon.

---
date: 2010-03-17 23:24:30+00:00
description: ''
excerpt: Heap allocation stack traces are useless when using certain versions of the
  MSVC runtime. Is it possible to modify and rebuild MSVCR80 to avoid this?
featuredImage: ''
slug: /modifying-the-vc-runtime-to-get-better-heap-allocation-stack-traces
template: blog-post
title: Modifying the VC runtime to get better heap allocation stack traces
categories:
- Debugging
- Software Development
- WinDbg
- Windows
tags:
- c++
- Debugging
- heap
- WinDbg
---

Today I was trying to track down some - how can I put this politely - "unusual" memory usage in some unmanaged code running inside Excel. I broke out WinDbg and tried the usual suspects to get an idea of how memory was being used. Unfortunately, the way that msvcr80.dll is built stopped me from getting decent stack traces for the allocations, so I decided to try and rebuild it with a fix to remedy the situation.
<!-- more -->


## Collecting stack traces


One of the most helpful things the heap manager can do for you when investigating memory issues is to capture stack traces for each heap allocation. You can enable the "collect stack traces" heap flag using the gflags GUI or from within WinDbg:

    
    
    0:006> !gflag +ust
    Current NtGlobalFlag contents: 0x00001040
        hpc - Enable heap parameter checking
        ust - Create user mode stack trace database
    


This means that for each heap block (one located at `0x0bbf7308` in this case), you can see where it was allocated by using the -a (show all information) option:

    
    
    0:006> !heap -p -a 0bbf7308 
        address 0bbf7308 found in
        _HEAP @ a630000
          HEAP_ENTRY Size Prev Flags    UserPtr UserSize - state
            0bbf7308 0073 0000  [07]   0bbf7310    00380 - (busy)
            Trace: 401c
            7c96d6dc ntdll!RtlDebugAllocateHeap+0x000000e1
            7c949d18 ntdll!RtlAllocateHeapSlowly+0x00000044
            7c91b298 ntdll!RtlAllocateHeap+0x00000e64
            78134333 MSVCR80!malloc+0x00000077
    


But the obvious problem with this is that the stack trace always stops at malloc. Something's allocating some memory? You don't say... 

It turns out that this is a [well understood](http://http://www.nynaeve.net/?p=209) and documented issue with the Microsoft VC++ runtime, variously known as msvcrt, msvcr70, msvcr71, msvcr80, msvcr90, etc. Unfortunately they're all built using the stack frame pointer omission optimisation. Well they're built with the [-O1](http://msdn.microsoft.com/en-us/library/8f8h5cxt.aspx) (favour small code) option, which implies [-Oy](http://msdn.microsoft.com/en-us/library/2kxx5t2c.aspx). This means that the fast stack-walking algorithm the heap manager uses stops at functions without a return address. The only way to get a decent trace in this situation would be to use the DbgHelp API along with the .pdb files, which would be far too slow to do at each allocation site.


## "Fixing" it


So, given that the source for the runtime library ships as part of Visual Studio, maybe it would be possible to build it without the -Oy option?

My first attempt at building it failed miserably with errors like:
`
NMAKE : fatal error U1073: don't know how to make 'build\intel\mt_obj\startup.lib'
`
Luckily this [excellent page](http://blogs.msdn.com/michkap/articles/478235.aspx) helped me get past this to a point where I could actually get a DLL built.

The next stage was to modify the build scripts to use different compiler switches. This was simply a case of changing line 69 of `makefile.sub` from:
`CFLAGS=$(CFLAGS) -O1`
to:
`CFLAGS=$(CFLAGS) -O1 **-Oy-**`

I thought I may have to also modify the build scripts to output a version of the DLL with the same name as the file I was replacing, msvcr80.dll, directly, in case there were internal references to the name in embedded manifests. There's a section at the top of the build script for choosing a name for your private version of the library, but it strongly discourages use of the "reserved" MSVC* names. Luckily it turns out not to be necessary; the DLL is constructed in such a way as to be rename-able without any ill effects. I could build _sample_.dll (the default output name) and simply copy it to the destination directory in the SxS tree (`C:\WINNT\WinSxS\x86_Microsoft.VC80.CRT_1fc8b3b9a1e18e3b_8.0.50727.3053_x-ww_b80fa8ca`) and rename it.



## Result


Now I get the expected full stack trace (names have been changed to protect the innocent):

    
    
    0:006> !heap -p -a 0bbf7308 
        address 0bbf7308 found in
        _HEAP @ a630000
          HEAP_ENTRY Size Prev Flags    UserPtr UserSize - state
            0bbf7308 0073 0000  [07]   0bbf7310    00380 - (busy)
            Trace: 401c
            7c96d6dc ntdll!RtlDebugAllocateHeap+0x000000e1
            7c949d18 ntdll!RtlAllocateHeapSlowly+0x00000044
            7c91b298 ntdll!RtlAllocateHeap+0x00000e64
            78134333 MSVCR80!malloc+0x00000077
            7816207f MSVCR80!operator new+0x0000001d
            fa92336 leakydll!std::allocator<std::vector<atl::cadapt<atl::ccombstr>,std::allocator<atl::cadapt<atl::ccombstr> > > >::allocate+0x00000016
            fa9879b leakydll!std::vector<ccomvariant,std::allocator<ccomvariant> >::resize+0x0000005b
            ...
            ...etc...
    


That'll make it _much_ easier to work out what's happening and who's responsible. Of course, you should be careful with this modified version. Only use it on development machines, and make sure it doesn't escape into the wild: with great power comes great responsibility.

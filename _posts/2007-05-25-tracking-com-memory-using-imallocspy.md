---
date: 2007-05-25 22:28:36+00:00
description: ''
featuredImage: ''
slug: /tracking-com-memory-using-imallocspy
template: blog-post
title: Tracking COM memory using IMallocSpy
categories:
- COM
- Software Development
---

 




 




One of my aims at work is to simplify the development and testing regime as much as possible, and this mostly consists of making sure we’re using the most appropriate tools and technologies wherever possible. In the context of correctness checking this involves determining whether the time we spend installing, configuring and chasing down false positives in third-party tools such as Purify outweighs their benefits.




As a rule I’d always favour using built-in, vendor provided hooks rather than a bolt-on product, and as such I was interested in what we could do with the IMallocSpy functionality in the COM runtime, as our group’s software is almost all COM based.




It turned up some interesting things…




The first was the behaviour of CoRevokeMallocSpy. Stupidly I originally neglected to check the return value, and then when I did, found it was returning E_ACCESSDENIED. It turns out that this means there are still allocations that occurred when the spy was active that haven’t been freed. Given that my use of the spy was to check for leaked allocations in the first place, this was a bit annoying; at least, it meant I couldn’t overload the lifetime of my spy object, to, say, dump a list of leaks when it was destroyed. If there were any leaks it never got destroyed!




The next problem was that it seemed whenever I had an app that called the apparently innocous CLSIDFromProgID, it would leak memory. For example:




    
    addr: 0x0015eb20:
    size: 0x2c (44)
    contents:
    c0 31 fd 76 30 c8 15 00 01 00 00 00 01 00 12 00 .1.v0...........
    28 00 10 00 00 00 00 00 00 00 00 00 00 00 00 00 ................
    01 f0 ad ba bc ec 15 00 94 ec 15 00             ............
    [0x77583315] CSpyMalloc_Alloc+0x49
    [0x774fd073] CoTaskMemAlloc+0x13
    [0x76fd18fb] operator new+0xe
    [0x76fd5c6b] StgDatabase::InitClbFile+0x2e
    [0x76fdc190] StgDatabase::InitDatabase+0x623c
    [0x770076fa] OpenComponentLibraryEx+0x3e
    [0x77005306] OpenComponentLibraryTS+0x1a
    [0x76fd954d] _RegGetICR+0x761f
    [0x76fd1f24] CoRegGetICR+0xffff877d
    [0x76fd6a20] IsSelfRegProgID+0x65
    [0x76fd80f9] CComCLBCatalog::GetClassInfoFromProgId+0x1783
    [0x77518a6d] CComCatalog::GetClassInfoFromProgId+0x100
    [0x77518964] CComCatalog::GetClassInfoFromProgId+0x1e
    [0x775188a0] CLSIDFromProgID+0x76
    [0x004120f5] wmain+0xa5
    [0x004173a6] __tmainCRTStartup+0x1a6
    [0x004171ed] wmainCRTStartup+0xd
    [0x7c816fd7] BaseProcessStart+0x23




Looking at the stack trace, it seemed there was some kind of internal caching going on, but what was confusing me was that I was under the impression that all memory allocated by the COM runtime would be freed by the time CoUninitialize was done. After all, you can’t make any further COM calls after this point. If you don’t believe me, just try using a static CComPtr, and see what happens in DllMainCRTStartup when your app exits.




After a bit of poking about with WinDbg (thank goodness we get decent symbols for the OS now), I could see that some kind of “database” was being created within CLBCATQ.DLL:




    
    0:000> k
    ChildEBP RetAddr¼br> 0012faf4 770076fa CLBCATQ!StgDatabase::InitDatabase
    0012fb18 77005306 CLBCATQ!OpenComponentLibraryEx+0x3e
    0012fb34 76fd954d CLBCATQ!OpenComponentLibraryTS+0x1a
    0012fdd0 76fd1f24 CLBCATQ!_RegGetICR+0x205
    0012fdf0 76fd6a20 CLBCATQ!CoRegGetICR+0x29
    0012fe48 76fd80f9 CLBCATQ!IsSelfRegProgID+0x6b
    0012fe88 77518a6d CLBCATQ!CComCLBCatalog::GetClassInfoFromProgId+0x51
    0012fec0 77518964 ole32!CComCatalog::GetClassInfoFromProgId+0x149
    0012fee0 775188a0 ole32!CComCatalog::GetClassInfoFromProgId+0x1e
    0012ff0c 00401340 ole32!CLSIDFromProgID+0x95
    0012ff7c 00401cae testleakcheck!wmain+0x60
    0012ffc0 7c816fd7 testleakcheck!__tmainCRTStartup+0x10f [f:\rtm\vctools\crt_bld\self_x86\crt\src\crtexe.c @ 583]
    0012fff0 00000000 kernel32!BaseProcessStart+0x23




I could see by looking at the exports that there was a function called CoRegCleanup in CLBCATQ.DLL that looked like it could be used to free up this storage before I did my leak checking. Calling it by dynamically getting the function pointer using GetProcAddress did make a difference, but there was still some memory not freed, and I didn’t feel comfortable using an undocumented function in this way.




Then I remembered the magical OANOCACHE environment variable.




This is used to tell the COM runtime not to cache memory used for BSTRs, VARIANTs, SAFEARRAYs, or anything else allocated using CoTaskMalloc. So, I set the variable, re-ran the test and voila! the apparent leaks disappeared. There must be something in CLBCATQ that detects the environment varibale and disables it’s internal cache.




So the moral of the story is; if you’re attempting to reliably track memory usage with IMallocSpy, remember to make sure you have OANOCACHE set, otherwise you will always end up with memory not being freed until late into process teardown.

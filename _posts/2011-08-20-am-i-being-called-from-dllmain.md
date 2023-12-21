---
date: 2011-08-20 12:52:37+00:00
description: ''
excerpt: How can you tell if your code is being called from within DllMain? You could
  use an undocumented function from ntdll.dll.
featuredImage: build/gatsby/www.ianvoyce.com/assets/2011-08-20-am-i-being-called-from-dllmain_lock_200.png
slug: /am-i-being-called-from-dllmain
template: blog-post
title: Am I being called from DllMain?
categories:
- .NET
- Debugging
- Visual Studio
- WinDbg
- Windows
---

[caption id="attachment_1265" align="alignright" width="200" caption="Lock; literal images \'r\' us"][![Lock; literal images 'r' us](http://www.ianvoyce.com/wp-content/uploads/2011/08/lock_200.png)](http://www.ianvoyce.com/wp-content/uploads/2011/08/lock_200.png)[/caption]While Googling for an obscure Windows function the other day, I came across [this](http://www.geoffchappell.com/viewer.htm?doc=index.htm) fantastically useful repository of undocumented WinAPI functions, put together by Geoff Chappell. I'm not sure how I hadn't discovered it before.

One of the functions that immediately caught my eye was [LdrLockLoaderLock](http://www.geoffchappell.com/viewer.htm?doc=studies/windows/win32/ntdll/api/ldrapi/lockloaderlock.htm). I'd previously spent quite a few frustrating hours trying to figure out how to determine whether some code was being executed from DllMain, i.e. while in the loader lock, so I could avoid doing anything dodgy - or indeed, anything at all. 

The case I was looking at was some logging library code that was used, amongst other things, to record the fact that DLLs were being unloaded. Unfortunately when this was called from DllMain, it sometimes caused a deadlock, for all the reasons we already know about. The library code was called from lots of DLLs, so it wasn't feasible to fix all of the call sites, instead I had to make the logging a no-op when it's not safe.
<!-- more -->
I'm embarrassed to say that my previous attempt to detect the lock involved some pretty heinous hackery. I worked out the memory address (the offset within ntdll.dll) where the loader lock critical section is located, cast that bit of memory to a CRITICAL_SECTION and tested it. I even had to provide the ability to change the offset based on the version of ntdll being used, in a vain attempt to reduce its fragility. Ouch. It was very nasty, and to be honest although it worked in the cases where I tested it, I was reluctant to release it.

Luckily, along comes LdrLockLoaderLock to save my blushes. It appears to give me exactly the functionality I need; you can pass a flag to tell it to return immediately if the lock's already been taken, and there's a status parameter that can be used to tell if you got the lock - whereupon you can call the corresponding [LdrUnlockLoaderLock](http://www.geoffchappell.com/viewer.htm?doc=studies/windows/win32/ntdll/api/ldrapi/unlockloaderlock.htm). Nice!  

I wonder if this is what's used by the Visual Studio [loader lock managed debugging assistant](http://msdn.microsoft.com/en-us/library/ms172219.aspx) to determine if your managed code is being run under the loader lock?

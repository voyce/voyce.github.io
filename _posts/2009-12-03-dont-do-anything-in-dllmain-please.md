---
date: 2009-12-03 23:27:13+00:00
description: ''
excerpt: Thinking of adding some code to your DLLs DllMain function? STOP!
featuredImage: ''
slug: /dont-do-anything-in-dllmain-please
template: blog-post
title: Don't do anything in DllMain... Please
categories:
- Debugging
- Software Development
- Windows
tags:
- c++
- COM
- Debugging
- dll
- win32
---

Novice Windows programmers can often think that `DllMain` is a good place to get that one-time set-up and tear-down work done. It seems to offer an ideal opportunity to know when your DLL has just been loaded, and when it's about to be unloaded. What better place to add all that expensive, complicated initialisation...? STOP! WAIT! Before you add anything in `DllMain`, make sure you understand what state the process will be in when it gets called. Once you know that, you may well change your mind...
<!-- more -->
Firstly, take a look at [this page](http://www.microsoft.com/whdc/driver/kernel/DLL_bestprac.mspx). It does a pretty good job of ramming home the point that there's very little that it's safe to do in `DllMain`. Essentially this is because while the function's being called, the OS is holding a process-wide lock that isn't re-entrant. As such, if you do anything that causes a DLL to be loaded, a deadlock may occur. There are many, many things that may have a side-effect of loading a DLL; calling COM functions, creating threads etc.

This is such a common source of bugs, and such an important requirement, that from Vista onwards Microsoft introduced a new set of functions in the Windows API explicitly to support it: [One-Time Initialization](http://msdn.microsoft.com/en-us/library/aa363808(VS.85).aspx).

And even if you get away with doing naughty things in `DllMain` now, don't think that it'll stay that way forever. We got away with it for years, then when .NET came along it introduced all sorts of additional correctness checks. For instance, the Managed Debugging Assistant (MDA) in Visual Studio will shout loudly should you attempt to run managed code during `DllMain`. 
`
Managed Debugging Assistant 'LoaderLock' has detected a problem in 'C:\YourApp.vshost.exe'.
Additional Information: Attempting managed execution inside OS Loader lock. Do not attempt to run managed code inside a DllMain or image initialization function since doing so can cause the application to hang.
`
And it's easier than you think to do so. For example, calling something as innocuous as `GetWindowText` can result in managed code being run. 

How can you get around this?

One of the approaches I've used has been to make use of Win32 asynchronous procedure calls. Specifically you can call [QueueUserAPC](http://msdn.microsoft.com/en-us/library/ms684954(VS.85).aspx) to add a function to the queue, and this can contain the initialisation you would've otherwise done in `DllMain`. 

However there is significant gotcha regarding use of APCs: your function will not be called until the thread is in an "alertable wait state". This means you (or some other code on the thread) need to call an alertable wait function such as [`SleepEx`](http://msdn.microsoft.com/en-us/library/ms686307(VS.85).aspx), [`WaitForSingleObjectEx`](http://msdn.microsoft.com/en-us/library/ms687036(VS.85).aspx) and specify TRUE for the alertable parameter. 

Once your APC is getting called successfully you'll be in much better place; your code will be executed outside of the scope of the dreaded OS loader lock and you'll be doing things by the book, hopefully avoiding all the potential pitfalls that lie in wait within `DllMain`.

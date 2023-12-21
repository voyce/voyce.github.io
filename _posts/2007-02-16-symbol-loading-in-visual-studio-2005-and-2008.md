---
date: 2007-02-16 22:35:44+00:00
description: ''
featuredImage: ''
slug: /symbol-loading-in-visual-studio-2005-and-2008
template: blog-post
title: Symbol loading in Visual Studio 2005 and 2008
categories:
- Visual Studio
---

I don’t know if I’ve been spoilt by the lazy symbol loading in WinDbg, but it seems incredibly slow to start up unmanaged processes under Visual Studio 2005. It spends a huge amount of time attempting to load symbols for every single DLL that gets loaded. As far as I can tell it doesn’t do any caching or recording of the fact that symbols aren’t available for specific binaries. It just blindly tries every time.


Anyway, I’ve found a way to improve matters slightly. It turns out you can use the symbol server DLL - symsrv.dll - from the most recent version of WinDbg (6.6.3.5 at the last check) to replace the version that ships with Visual Studio 2005.

One of the additional features that the WinDbg version offers in the exclusions list. You can use this to avoid loading symbols for anything other than the DLL under test. Simply copy the DLL into C:\Program Files\Microsoft Visual Studio 8\Common7\IDE and create a symsrv.ini file with the following contents:

[exclusions]
*.*

This should make your symbol loading fly… yet it still loads the symbols required to debug as normal. Magic!

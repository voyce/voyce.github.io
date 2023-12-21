---
date: 2009-06-03 09:23:37+00:00
description: ''
featuredImage: ''
slug: /windbg-locks-command-broken
template: blog-post
title: WinDbg !locks command broken
categories:
- Debugging
- WinDbg
tags:
- Debugging
- WinDbg
---

It seems that the _extremely_ useful `!locks` command is broken in 6.11.1.40x, the current and previous release of WinDbg from the [debugging tools for Windows](http://www.microsoft.com/whdc/DevTools/Debugging/default.mspx).

You'll get errors like:

    
    
    0:007> !locks
    NTSDEXTS: Unable to resolve ntdll!RTL_CRITICAL_SECTION_DEBUG type
    NTSDEXTS: Please check your symbols
    



The suggested solution seems to be to roll-back to version 6.10.3.233, available from [here](http://msdl.microsoft.com/download/symbols/debuggers/dbg_x86_6.10.3.233.msi), or you can just replace the version of `ntsdexts.dll` in the `c:\program files\debugging tools for windows (x86)\winxp` directory with the one from the earlier release.

Judging by the error message, I'm guessing that the new version may work if you happen to be using a debug (checked) build of the Windows kernel, but I don't have one around to try it with.

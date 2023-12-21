---
date: 2012-09-24 22:37:09+00:00
description: ''
featuredImage: build/gatsby/www.ianvoyce.com/assets/2012-09-24-examining-pdb-files-with-dbh_breakpoint2.png
slug: /examining-pdb-files-with-dbh
template: blog-post
title: Examining PDB files with DBH
categories:
- Debugging
- Software Development
- Visual Studio
- WinDbg
tags:
- dbghelp
- dbh
- Debugging
- pdb
- symbols
- WinDbg
---

Wow, it's been a ridiculously long time since I've blogged. I think it's time I put something up just to break the curse, and this seemed like a good, and hopefully useful, place to start. Time to polish some of these dusty drafts into published gems.

[![](build/gatsby/www.ianvoyce.com/assets/2012-09-24-examining-pdb-files-with-dbh_breakpoint2.png)](build/gatsby/www.ianvoyce.com/assets/2012-09-24-examining-pdb-files-with-dbh_breakpoint2.png)Ever been in that situation where you (or someone else) finds that Visual Studio just won't set a breakpoint in some source code that you're _sure_ should be being used? You'll see the hollow breakpoint icon and something like 'The breakpoint will not currently be hit. No symbols have been loaded for this document'.
<!-- more -->
The first, and most obvious, step is to make sure that a PDB file is actually being loaded at all. Look in the modules window and check that Symbol Status is Symbols Loaded and there's a symbol file listed. Assuming there is, how can we dig a little deeper and find out what the PDB file actually contains? Remember: the PDB will only contain symbols for code that's actually used in your binary, so that will exclude any dead code stripped out by the linker.

Of course, there are a couple of existing mechanisms to examine PDBs: you could write something yourself using the [Debug Help Library](http://msdn.microsoft.com/en-us/library/windows/desktop/ms679309(v=vs.85).aspx) (aka dbghelp) directly - a little too labour-intensive for this case - or you could use the `x` (eXamine symbols) command in WinDbg (or in the immediate window of Visual Studio). E.g. this will dump all of the symbols (probably a lot of them) in module testapp:

    
    x testapp!*
    


Of course, in order to use a debugger to do this, you'd have to have a crash dump or live session with the DLL loaded.

Wouldn't it be cool if there was some way of examining the PDBs offline? Well, it turns out there is, it goes by the name of [DBH](http://msdn.microsoft.com/en-us/library/windows/hardware/ff540437(v=vs.85).aspx), it ships alongside WinDbg in the Debugging Tools For Windows package and it's incredibly useful. It works in one of two modes: either an interactive REPL or batch mode with a single command passed as an argument.

Its target can be a running process, the PDB file itself, or a DLL file. In the latter case, and assuming you've got _NT_SYMBOL_PATH properly set-up, it will identify and download the symbols automatically based on the DLL identity. With the "noisy" option enabled, we can see what it does in a simple case where symbols are co-located: 

    
    
    C:>dbh -n testapp.exe
    verbose mode on.
    DBGHELP: No header for testapp.exe.  Searching for image on disk
    DBGHELP: C:\Users\Administrator\dev\testapp\Debug\testapp.exe - OK
    DBGHELP: testapp - private symbols & lines
             C:\Users\Administrator\dev\testapp\Debug\testapp.pdb
    



To dig deeper, let's look inside the file for some destructors, i.e. all symbols containing a tilde:

    
    
    C:>dbh testapp.pdb enum *~*
    
     index            address     name
         1            1011400 :   testClass::~testClass
    



But for the particular scenario we're discussing, we'd like to see which source files are indexed by the PDB, rather than which symbols. You can do that with the `src` option on the command line:

    
    
    C:>dbh testapp.pdb src
    ...
    c:\users\administrator\dev\testlib\testlibclass.h
    c:\program files (x86)\microsoft visual studio 10.0\vc\include\vadefs.h
    ...
    f:\dd\public\sdk\inc\winnls.h
    ...
    


Here I've shown some of the files that highlight the fact that the PDB file contains the source that we built, the VC headers that are compiled into it, _and_ the headers that have been compiled into the version of the CRT that we're using.

So, now, given a compiled DLL and it's PDB file, we can quickly answer the question "why can't I set a breakpoint in file X?" by checking that the source file is actually included in the PDB.

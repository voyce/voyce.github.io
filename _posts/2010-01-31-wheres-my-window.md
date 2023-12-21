---
date: 2010-01-31 20:06:30+00:00
description: ''
excerpt: Where's my window gone? I'm sure I left it around here somewhere...
featuredImage: ''
slug: /wheres-my-window
template: blog-post
title: Where's my window?
categories:
- Windows
tags:
- c++
- hidden
- invisible
- missing
- screen
- win32
- Windows
---

On Windows, if you regularly change screen resolution or size, perhaps by accessing a machine remotely, you might find some of your application windows are no longer visible; they're positioned outside of the visible display area. If you can't see the window, it can be a _little_ difficult to use the application. How can you get that window back?
<!-- more -->
In most cases - but only only on pre-Vista versions of Windows - it can be easily resolved by right clicking on the task bar icon for the application, selecting Move and then using the cursor keys. This makes the cursor "stick" to the caption of the window, and you can move the mouse (without clicking!) to bring it onto the screen. Clicking releases the window.

You can do it without the mouse, too. Just make sure the window is selected in the task bar (perhaps by using the Windows key and then tabbing to the appropraite icon), then hit Alt-Space, Alt-M, and then using the cursor keys.

The only problem is, some application developers choose to change the system menu - the menu visible from clicking the app's icon in the top left, or by right-clicking the task bar icon - perhaps deciding for some insane reason not to include the standard Move option (as an aside, this breaks one of the fundamental tenets of usability: don't change - or in this case, remove - existing, established behaviour). If this is the case, you can instead use a handful of lines of Windows API code to access the window and move it programatically. Here's the entire code of a program that will do exactly that:

    
    
    #include <windows.h>
    #include <tchar.h>
    
    int _tmain(int argc, TCHAR *argv[])
    {
    	if (argc < 2)
    	{
    		_tprintf(_T("usage: mover windowtitle\n"));
    		return -1;
    	}
    	HWND hwnd = FindWindow(NULL, argv[1]);
    	if (hwnd)
    	{
    		SetWindowPos(hwnd, HWND_TOP, 0, 0, 400,400, 0);
    	}
    	else
    		_tprintf(_T("Unable to find window %s\n"), argv[1]);
    	return 0;
    }
    


Compile it up and you can use it as a handy little utility to move arbitrary top level windows, e.g. if you've got regedit running:
`
mover "Registry Editor"
`
The only tricky thing is finding the exact title of the window to use. Well, you wouldn't want it to be _too_ easy, would you?

---
date: 2009-08-21 20:57:12+00:00
description: ''
featuredImage: build/gatsby/www.ianvoyce.com/assets/2009-08-21-largest-free-block-of-address-space_buildings_small.png
slug: /largest-free-block-of-address-space
template: blog-post
title: Finding the largest free block of address space
categories:
- Software Development
- WinDbg
- Windows
tags:
- address space
- c++
- perfmon
- private bytes
- win32
- WinDbg
---

[![Buildings](http://72.47.193.211/wp-content/uploads/2009/08/buildings_small.png)](http://72.47.193.211/wp-content/uploads/2009/08/buildings_small.png)I've been seeing problems recently with fragmented virtual address space. During the lifetime of a process, bits and pieces of memory are allocated throughout the 2GB 32-bit address space to such an extent that large contiguous blocks of free space are no longer available. If anything subsequently requires a large block of memory (like, for instance, a somewhat out-of-date version of the GHC runtime), it will fail to get it.

It's obvious looking at the output from [VMmap](http://www.ianvoyce.com/index.php/2009/07/28/diagnosing-out-of-memory-errors-with-vmmap/) or windbg's `!address` command what the largest contiguous block is, e.g.

    
    
    0:008> !address -summary
    ....
    Largest free region: Base 07300000 - Size 63ed0000 (1637184 KB)
    


But what if you need that number in order to make a decision at run-time? For instance, to decide whether your process is in a fit state to continue, or if it should instead commit [hara-kiri](http://en.wikipedia.org/wiki/Seppuku). In that case, you need to access the information programmatically. That's where the immensely useful [VirtualQueryEx](http://msdn.microsoft.com/en-us/library/aa366907(VS.85).aspx) function comes in...
<!-- more -->
VirtualQueryEx gives you information on a single page of your virtual address space at a time. Pages size are dependent on the architecture and OS, but if you just want to iterate over all of them, you don't need to add any special handling; the function returns the size of the page in an element of the [MEMORY_BASIC_INFORMATION](http://msdn.microsoft.com/en-us/library/aa366775(VS.85).aspx) structure, so you can simply move to the 'next' page regardless of size.

If you're interested in free space, you'll need to find all the pages that have a state of MEM_FREE (0x10000), and that's pretty much all there is to it. By keeping track of how much space falls into a continuous range of MEM_FREE pages you can get to the number reported by VMmap and `!address`.

Here's some C++ code that returns the address of the largest free contiguous block in `largestFreestart`largestFree`. Enjoy!


    
    
    	MEMORY_BASIC_INFORMATION mbi;
    	DWORD start = 0;
    	bool recording = false;
    	DWORD freestart = 0, largestFreestart = 0;
    	__int64 free = 0, largestFree = 0;
    	while (true)
    	{
    		SIZE_T s = VirtualQueryEx(hproc, reinterpret_cast<lpvoid>(start), &mbi, sizeof(mbi));
    		if (s != sizeof(mbi))
    		{
    			if (GetLastError() != ERROR_INVALID_PARAMETER)
    				return ReportError(GetLastError(), _T("Failed to VirtualQueryEx at %08x"), start);
    			else
    				break;
    		}
    		if (mbi.State == MEM_FREE)
    		{
    			if (!recording)
    				freestart = start;
    			free += mbi.RegionSize;
    			recording = true;
    		}
    		else
    		{
    			if (recording)
    			{
    				if (free > largestFree)
    				{
    					largestFree = free;
    					largestFreestart = freestart;
    				}
    			}
    			free = 0;
    			recording = false;
    		}
    		start += mbi.RegionSize;
    	}
    

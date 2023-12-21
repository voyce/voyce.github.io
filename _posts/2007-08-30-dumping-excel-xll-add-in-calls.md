---
date: 2007-08-30 23:40:55+00:00
description: ''
featuredImage: ''
slug: /dumping-excel-xll-add-in-calls
template: blog-post
title: Dumping Excel XLL add-in calls
categories:
- Debugging
- Excel
- WinDbg
---

Using WinDbg it’s possible to get a dump of each XLL call that is made by Excel as it calculates. If you’re using Excel 2003, create the following breakpoint that dumps the symbol at eax+4 (the entry point that is about to be called), then continues.:


`bu EXCEL!MdCallBack+0xa880 "dds @eax+0x4 L1; g"`

You’ll need to adjust the offset for other versions of Excel - and I haven’t tried it yet with 2007. Assuming you’ve got symbols available for the XLLs being called, you’ll get something like this:

`0013bc50 15109730 addin1!addin1_function1
0013bc50 12c0e918 addin2!addin2_anotherfunctions`

This technique can be useful when troubleshooting - to identify the last addin call being made before a failure perhaps - and is also quite interesting to just watch and see the pattern in which your XLL UDFs get called.

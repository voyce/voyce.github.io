---
date: 2009-04-14 09:00:22+00:00
description: ''
featuredImage: ''
slug: /windows-sdk-breaks-fsharp-integration
template: blog-post
title: Installing Windows SDK breaks F# Visual Studio integration
categories:
- F#
- Visual Studio
tags:
- crash
- F#
- intellisense
- Visual Studio
- winsdk
---

Beware! If you install the [Windows SDK](http://www.microsoft.com/downloads/details.aspx?FamilyId=F26B1AA4-741A-433A-9BE5-FA919850BDBF&displaylang=en) - perhaps to get access to the interesting looking [WPF performance tools](http://blogs.msdn.com/jgoldb/archive/2008/09/25/updated-wpfperf-performance-profiling-tools-for-wpf.aspx) - you'll find that it hoses your F# Visual Studio integration. I found that it causes intellisense tooltips to stop appearing, and the integrated F# interactive to crash Visual Studio. Both of these issues are a real pain; especially the inability to see the inferred types "live", which is pretty much essential for F# development - where the focus is on compile time correctness.

I remembered seeing a [post on that Windows SDK blog](http://blogs.msdn.com/windowssdk/comments/7850578.aspx) that I'd come across relating to a similar issue with the XAML editor (I've been doing some work with WPF recently, more on that in a later post) so thought I'd try the steps they recommend, in short, re-registering TextMgrP.dll:

`regsvr32 "%CommonProgramFiles%\Microsoft Shared\MSEnv\TextMgrP.dll"`

...and all my problems went away. Hope you find this useful.

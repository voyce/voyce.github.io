---
date: 2007-03-28 22:45:35+00:00
description: ''
featuredImage: ''
slug: /boost-causes-visual-studio-2005-to-crash
template: blog-post
title: Boost causes Visual Studio 2005 to crash
categories:
- Visual Studio
---

After experiencing some nasty crashes in Visual Studio 2005 recently, I’ve discovered that the long, heavily templated type names in version 1.33.1 of the boost headers don’t play well with the internal buffer sizes in various parts of Visual Studio (including SP1). This results in crashes while generating Intellisence data (feacp.dll) and in the debugger itself. It’s fairly easy to reproduce, essentially all you have to do is #include [adjacency_list](http://www.boost.org/libs/graph/doc/adjacency_list.html) from the graph part of the boost library. I’ve raised a support incident with Microsoft, so we’ll see what they have to say about it.

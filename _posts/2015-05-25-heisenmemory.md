---
date: 2015-05-25 11:58:45+00:00
description: ''
featuredImage: ''
slug: /heisenmemory
template: blog-post
title: Heisenmemory
categories:
- .NET
- C++
- Debugging
- Rant
---

I am sick of non-deterministic memory management. So sick. This was the big promise of .NET wasn't it, way back in the 00s? No more having to worry about reference counting, double-frees or leaks. Well all we've done is switched it for worrying about event handlers, garbage collector pauses and weak dictionaries. 

If you don't have to worry about memory in .NET applications, then why are there so many commercial tools for solving memory-related problems? I'm starting to yearn for the days of being able to put an instance on the stack for the length of a curly-bracketed scope, and knowing that after that, it's gone.
<!-- more -->


### I can see your underwear


And it's not just the tooling that gives it away. The language and framework has a load of memory-specific artefacts sprinkled all over it: IDispose, finalizers, GCHandles and WeakReferences, to name but the classics. Recently, in an attempt to appease the GC gods - and brave souls writing games in .NET - version 4.5 has added latency modes to further complicate matters.  

Part of the problem is that there are so many factors that go into determining when your memory gets freed. Are you running the workstation garbage collector, or the server version? Have you enabled concurrent collection? And when does it even happen? Try and find something on MSDN that lists the explicit points where the .NET framework will attempt to reclaim memory.  

At least nowadays the GC can reclaim space on the large object heap. For an embarrassingly long time, it was possible to allocate memory from there (normally without even realising) and then never being able to defragment it!



### Invoking GC is a code smell


Whenever I see code that invokes the garbage collector explicitly, it makes me shiver. This is almost always the result of someone asking "oh, wow, look in Task Manager, why is my app using so much memory?". And calling GC.Collect is almost never the right answer. Maybe a superficially easy one, but not the right one. Face it: you're going to have to take a long hard look at your code and figure out what's going on. 

I guess the real issue is that you may not have to worry *so much* about your memory, but you damn sure still need to understand what's being done with it. There is no substitute for understanding your own code, if not the runtime. Hopefully, attempts to understand GC behaviour will be helped by the fact that [full code](https://github.com/dotnet/coreclr/blob/master/src/gc/gc.cpp) is now available for all to see. I say "all", but there can only be a handful of people who'll be able to wade through the 37000 line(!) source file and make head or tail of it.

This rant was prompted by reading about the appearance of Rust 1.0 recently. With modern languages like that and C++ 14 it looks like we're getting closer to enabling abstraction while maintaining control. About time too...

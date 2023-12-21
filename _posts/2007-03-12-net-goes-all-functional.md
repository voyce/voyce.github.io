---
date: 2007-03-12 22:44:40+00:00
description: ''
featuredImage: ''
slug: /net-goes-all-functional
template: blog-post
title: .NET goes all functional
categories:
- .NET
- Software Development
---

It’s interesting to watch how C# has developed over the past few years, becoming more and more like a functional programming language, with a [declarative](http://en.wikipedia.org/wiki/Declarative_programming) style favoured over the imperative approach of previous generations. This is especially obvious in C# 3.0, with LINQ borrowing and building on some technologies directly from the world of functional programming; e.g. lambda expressions, type inference and anonymous types.


It’s especially interesting for me, because where I work there is a lot of discussion of Excel as a functional programming language. For good or bad, Excel is one of the most heavily used “development” platforms used by people in banking and finance. And most of those people don’t think of Excel (and user defined functions within it) as a side-effect-free, functional language, although this is essentially what it is.

Having said that, most of the “real” development work that I do is still in C++, and it was quite a switch moving back even from some of the C# 2.0 I’ve been doing lately, with its anonymous delegates and generics, to what suddenly seemed like an incredibly verbose set-up in the C++ STL collections. It will be interesting to see how the C++0x lambda expression support is implemented, but to be honest I don’t think it’s enough to convince me; there are still too many complexities in C++ for it to be as productive as C#/.NET for the “average” developer.
It’s plain to see that the declarative approach is much more where we want to be as developers in a world of multi-core and many-cpu computing. At that point I really want the language to help me think about my algorithms at a much higher level; and let concurrency concerns be dealt with as automatically as possible, perhaps thanks to restrictions the language has placed on me such as my functions not having any side effects. As anyone who’s written (or studied) complex multi-threading code will know, it’s virtually impossible for human beings to deal with concurrency as a thought exercise. And in fact, even finding bugs in actual implementations is difficult because of the non-deterministic nature of multi-threading bugs.
It will be interesting for me to see how the multi-threaded calculation tree in Excel 2007 impacts overall spreadsheet calculation speed. It looks like the people at Microsoft have taken advantage of some of the functional aspects of Excel (even though they haven’t got as far as [some people](http://research.microsoft.com/%7Esimonpj/) would like) and married it with a somewhat concurrent implementation. Now all I have to do is rewrite those hundred or so XLLs….

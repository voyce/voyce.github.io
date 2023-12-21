---
date: 2008-01-22 22:55:58+00:00
description: ''
featuredImage: ''
slug: /the-truisms-of-software-development
template: blog-post
title: The Truisms of Software development
categories:
- Software Development
---

One of the things I remember vividly from various software development books is the fact that it is easier to write code than read it. This tends to mean that developers are always tempted to re-write applications in order to understand them; it’s notionally “easier” than trying to grok the code by inspection only.


The major problem with this approach lies in what you don’t see in the code. These are all the subtle workarounds and accumulated knowledge that exists almost between the lines. Unless the code is extremely well commented, and the developer actually takes the time to read and understand the comments, any rewriting, or even simply “cleaning-up” the code will almost certainly miss these subtleties.

You can see this in practice in a lot of companies when new developers arrive and, tutting at the weirdnesses in the existing codebase, set about rewriting things in their own image. This isn’t just an ego thing, it’s a fairly natural response to an unfamiliar codebase.

The problem is, despite being aware of all this, I keep doing it myself.

The latest example was taking an (admittedly nasty) combination of shell scripts, javascript and C++ and rewriting some of it completely while refactoring the rest. Although there were many up-sides to reimplementing the shell and javascript parts of it in .NET - better error handling and messages, richer functionality and performance improvements - it has taken much longer than expected. Much, _much_ longer than I expected. The problem was that, although it was me who wrote most of the code in the first place, even I had forgotten some of the subtleties of the way it worked.

Maybe next time I’ll heed my own advice.

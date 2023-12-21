---
date: 2008-09-24 15:27:27+00:00
description: ''
featuredImage: ''
slug: /static-libraries-are-evil
template: blog-post
title: Static libraries are Evil
categories:
- COM
- Excel
- Rant
- Software Development
tags:
- c++
- dll
- lib
---

In my opinion.

Why? Well, because it's too easy to use them as an excuse for not defining your shared library interfaces properly.

The reason this is on my mind recently is that several hundred, yes, you heard that right, several _hundred_ DLLs have been released by my group over the last, ooh, 10 years or so. They are all still in use. Each of them has burned into it a copy of the library that deals with interfacing with Excel. That means each of these has it's own little internal copy of the current state-of-the-art. The problem with that is; the state-of-the-art moves on. And how do you go about updating the DLLs that are already in production? You have to re-release them. In an environment where thes DLLs are used for marking the profit and loss on a large derivatives trading book, that's not a small undertaking. And it's made worse if, say the DLL in question was last released with a different version of the compiler.

My approach would be to refactor this shared static library (.lib) into a stand-alone DLL.

At this point, people start saying "oh, but then you've got a single point of failure, if you release a broken version of that DLL, everything will stop working!". Not exactly a compelling argument. If the functionality of the DLL is well defined, and there are well known entry points it should be easy to put together a comprehensive black-box test suite. In fact we already do that with all our other DLLs (COM servers). The fact that this shared library *isn't* a DLL has meant that it's fallen through the testing cracks; another good reason to refactor it.

The internal interface to the shared library is already relatively well defined. It has a set of header files that define all of the functions and classes that are consumed by others. It's a relatively small step to compile it as a DLL, rather than a static library. The problem then becomes one of maintenance, dealing with the inevitable changes to the external interface in a backwardly compatible way.

And that's the problem. It requires some effort. Elsewhere in our codebase we use COM as a magic cure-all for avoiding having to deal with versioning: interface immutability rules. All interfaces are public, no published interface ever changes, object identity is based purely on interfaces supported. If you haven't got these crutches to rely on, then you have to enforce the rules yourself, which can be both logistically and technically difficult when you're dealing with C++.

But it's not impossible. And I really think it would be better than having hundreds of DLLs all containing subtley different versions of the same code, and being unable to change behaviour across the board without having to build, test and release them all.

Maybe you've got a different opinion?

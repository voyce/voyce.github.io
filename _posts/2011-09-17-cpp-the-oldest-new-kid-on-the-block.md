---
date: 2011-09-17 23:06:09+00:00
description: ''
excerpt: Is C++ coming back to a warm welcome from Microsoft?
featuredImage: build/gatsby/www.ianvoyce.com/assets/2011-09-17-cpp-the-oldest-new-kid-on-the-block_ballmer_cpp.png
slug: /cpp-the-oldest-new-kid-on-the-block
template: blog-post
title: 'C++: The oldest new kid on the block'
categories:
- C++
- Rant
- Software Development
- Windows
tags:
- .NET
- c++
- Visual Studio
- win32
---

[![Tasty](build/gatsby/www.ianvoyce.com/assets/2011-09-17-cpp-the-oldest-new-kid-on-the-block_ballmer_cpp.png)](build/gatsby/www.ianvoyce.com/assets/2011-09-17-cpp-the-oldest-new-kid-on-the-block_ballmer_cpp.png)Nobody could have failed to notice the recent resurgence of interest in the C++ programming language. In particular, the recent [Build](http://www.buildwindows.com/) conference was the most we’ve seen Microsoft talking about C++ for several years. Why has a language that’s been languishing in the “prehistoric irrelevance” category for so long suddenly come back into vogue?
<!-- more -->


### Dark Days


We should bear in mind that in some camps, C++ never went away. The financial, scientific and games programming communities have bravely kept the fires burning.
 
Why? Primarily performance. In these industries performance is a feature, it’s a key differentiator, and it will help you make money. Can you say the same for an enterprise app (there I go, using that term disparagingly again) that just displays a form interface over a relational database?

However, even in the C++ bastions, the language was getting a bad press. I can only speak for the finance industry, where I’ve been working for the past 10 years, but even here the focus briefly switched away from raw performance to complex (overly-complex, dangerously-complex, as hindsight tells us) trades where getting to market first with something new was key: developer productivity was key. Flexibility and generality was key.

In that case, you don’t want to be dealing with the vagaries of subtle errors, crashes and memory leaks that can easily creep in when you’re using a language as low-level as C++. You just want to be able to release quickly. You probably want to be using a managed language.

In fact, C++ was getting a bad press all over. The functional power that was provided in languages like Haskell and F# - and being hastily added to C# - was missing or awkwardly implemented in C++. The provision of things like the .NET thread pool provided programmers on the CLR an easy way to schedule asynchronous tasks, and it was being touted as the only way to survive in the highly-parallel near-future. Of course, it was possible to do this stuff in C++, it was just more difficult.   

Despite this, the programming community widely understood the power that C++ brings, but also that it has to be wielded carefully. Witness the oft-repeated jokes about the fact that when you shoot yourself in the foot with C++, you tend to blow your leg off. There were even tensions within Microsoft, with the pro-managed DevDiv vs the hardcore, native-favouring Windows division.


### The New Order


So what’s changed? Herb Sutter’s recent [talk](http://channel9.msdn.com/posts/C-and-Beyond-2011-Herb-Sutter-Why-C) covered many of the reasons why C++ is relevant again, and the most obvious reason is the emergence and importance of mobile devices and data centres. 

They have a completely different set of requirements than the desktop: they focus on squeezing the best experience out of every available cycle (and corresponding minute of battery life), and the most performance out of every watt of data centre power per degree of cooling. We’re seeing the impact of the smartphone in your pocket and the data-centric web and social media reflecting back onto the languages we use to write software.  

Managed languages fall down in some important areas here. They sacrifice memory efficiency to the god of garbage collection and they offer fewer opportunities for aggressive compiler optimisations, including, in the case of .NET, making use of chip-provided performance features like SIMD instructions.

Nowadays in more and more of the computing industry, it’s the hardware that’s driving the development choices, not the other way around.

With resource-constricted platforms, you want to be doing more of the heavy lifting offline, at compile time, not at run-time with a JIT compiler.


### How the iPhone showed devs don't care


While (parts of) Microsoft were busy pushing the managed languages hard, Apple was quietly producing game-changing pieces of hardware that the device-carrying public were buying in their hordes. And they weren’t just buying devices, they were buying apps to play on these devices - in their millions. Lots of developers wanted a piece of this action but - quelle horreur! - You had to use an obscure variant of C to create them.

But - surprise, surprise - developers did it anyway. Luckily, they weren’t on their own, but this time rather than a managed runtime, they had an API to use. And they had to use it; it was, and is, literally a condition of sale that your app uses only the APIs that Apple provide and in the way that they intended. Whatever your view on the fairness of this approach, having a restricted surface area and well defined set of libraries helps developers win back some of the productivity they might otherwise have sacrificed by using Objective-C.

Microsoft must’ve been looking askance at this, wondering how Apple had so effectively switched the model around. MS had been based on the idea of developer productivity being king and the hardware being utterly insignificant. Now developers were writing code in a language that was broadly derided by people in the programming community, yet they were in a virtuous circle where writing apps helped to ship hardware that increased the market for their apps. 


### We need libraries


Is it possible to find a good balance between the raw power of C++ and the productivity of managed languages? I think it is; and partly it comes down to having a good set of libraries. When I say good, I mean consistent, relevant, well-documented, easily available set of libraries. Can you see where C++ may have struggled here before?

For the low-level, programming task oriented basics, Boost is the answer. Seriously, Boost always seems to be the answer to questions of “how do I do X in C++?” where X is a modern programming technique. Smart pointers, lambdas, higher-order functions… they’re all there. 

A few years ago if you were working in C++, you didn’t have many options apart from vanilla STL, Boost, and writing-it-yourself (again). Now, the STL and the language itself is richer and anecdotally, organisations are more receptive to the use of Boost. So hopefully, this combination gives us back access to most of the functionality that’s available to managed developers in their pre-packaged base class libraries.


### C++ and Apps


But how is that going to help people use it to produce compelling apps and “experiences” (cough) and specifically, how is Microsoft going to use its developer productivity nous to keep Windows relevant in the new hardware markets? 

They need something else; something like the Apple Core frameworks. It looks like this is what Microsoft is talking about with Windows 8 and the Windows Runtime. It’s suddenly not embarrassed to admit that, hey, Windows itself is written in C++, so maybe you shouldn’t have to go through a managed ‘interpretation’ of the existing APIs, one that’s exclusively available to managed languages.

Of course, Microsoft can’t do anything as radical as forcing all developers targeting a mobile or tablet platform to use C++. For them, programmers, rather than any particular hardware, are the bread-and-butter, so they tend to have a slightly less dictator-style relationship with devs than Apple does. 

Judging from what I’ve seen from Build, Microsofts chosen approach is to provide language-specific “projections” (their term) from the Windows Runtime layer into a variety of languages, including C++. The native C++ types are mapped into corresponding concepts in the vernacular of the higher level languages; Javascript, C# etc. They’ve gone to some lengths, it seems, to ensure that transitions between levels are efficient. Using C++ at both levels should be the fast path, and of course you can mix it freely with code that, for example, uses Boost or other C++ libraries. 

Additionally and importantly, as well as basic types, collections etc, the Windows Runtime also contains all of the rich APIs for accessing app-level functionality; things like image capture, sound and geolocation. So suddenly, all of this stuff is available in all its glory directly from C++ - and other languages too via cheap bridges. What they’ve essentially got is an identical copy of the architecture that exists in the iOS world, while still attempting to keep non C++ developers on-side. 


### The Return...?


So, have Microsoft managed to bring C++ in from the cold? They seem to have admitted, at least to themselves, that developers will need to use it in order to effectively target some platforms, and they’ve used their tooling experience to make it look and feel like a managed language in their IDEs, as well as leveraging their compiler technology to do some additional useful performance tricks. It feels like C++ was a fundamental concern in the redesign of the new alternative to the creaking Win32 API.

But will the transition be something to make current developers switch to C++, especially those who’ve been used to living in the cosseted managed world? Perhaps, if the Microsoft-based tablet market becomes as compelling - read, large - as Apple’s. Otherwise I don’t see any reason why developers who don’t have specific platform or other runtime requirements would move; for the majority of environments, developer time is still the dominant cost, so developer productivity will still be paramount. 

The gap between managed and unmanaged languages has certainly narrowed in that area, but it still exists. C++ is still proving surprisingly relevant, despite its 30+ years, and hopefully its use in new and different areas of computing will feed back into improvements in the language and libraries for everyone’s benefit.

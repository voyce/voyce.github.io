---
date: 2012-01-04 23:07:47+00:00
description: ''
excerpt: A quick look back at some of the stuff I've been doing this year.
featuredImage: ''
slug: /looking-back-at-2011
template: blog-post
title: Looking back at 2011
categories:
- F#
- iOS
- Software Development
- WPF
tags:
- .NET
- '2011'
- F#
- iOS
- kinect
- Visual Studio
- WinDbg
---

Well, we're a few days into 2012 and no armageddon yet, so it's probably safe to take a quick glance back over our shoulder at some of the technical stuff that's flashed past in the preceding 12 months.
<!-- more -->


## Work: Using F# in anger


During working hours (and sometimes beyond) I've been developing a large project using lots of F#, a little bit of C# and a bit too much XAML. It's quite an interesting mix, with technologies that have a vastly different outlook on life: F# with it's strong, compile-time type checking and XAML with its, err, not-so-strong type checking. In fact, XAML and WPF play so fast and loose with types, that errors in the binding mechanism only happen at run-time and are almost always eaten silently. As you can imagine, this makes for some fun debugging. All I can say is thank goodness for [Snoop](http://snoopwpf.codeplex.com/).

We have a quite a substantial amount of ViewModel code in F# (tens of thousands of lines), much of it using `Async` to allow us to create responsive UIs without requiring tons of boilerplate code. Generally it works nicely, but I do have some concerns about the amount of thread hopping going on, and how that will affect performance. Especially as we break things down into pretty small pieces of functionality, sometimes just a few simple operations, and each is run on the thread pool. I don't have any concrete figures to go on here, and I'm hoping we don't end up in a situation where we sacrifice user productivity for developer productivity. Time (and perfmon) will tell.


### Debugging


I've done less hardcore WinDbg stuff this year. In fact, a couple of times I've been guilty of jumping in with the debugger where it probably would've been more effective to take a high-level look first. A case of having a debugger-shaped hammer, and everything looking like a nail.


### Moving to 64bit


A very positive thing that happend this year was the move to 64-bit Windows 7 for development at work, meaning that Visual Studio actually stays up without crashing for more than an hour. Previously I'd been having a terrible time: the combination of a couple of large-ish F# projects and 32-bit XP caused frequent VS out-of-memory errors, which was very frustrating. Luckily the move to Windows 7 has resolved it, and I can even run more than a handful of apps at the same time; what a luxury. 


## Home: iOS


After a long day at work, I finally get to go home, put the kids to bed and then my feet up, and break-out Xcode for a bit of iOS hacking. This year I've been feeling like I'm finally getting my head around some of the things that are fundamental in iOS development; from Objective-C concepts like release/retain semantics to iOS specifics, view controllers, table views etc. Previously I've been guilty of some copy-and-paste coding, mostly because it's pretty slow going getting up to speed with new tech (and the myriad iOS APIs) when you're only getting to do it for a few hours while knackered after work. Well, that's my excuse anyway.

I've been trying to finish off some of the apps that I've started, but not shipped. I tend to procrastinate a bit (massive understatement), and of course things aren't helped by the development 'fat tail', where the last 20% - polishing and getting things 'just right' - takes at least 80% of the time. My 2010 post on [Core Animation flip clocks](http://www.ianvoyce.com/index.php/2010/04/10/creating-an-ipad-flip-clock-with-core-animation/) continues to get a lot of traffic, and I finally bit the bullet and posted a (free!) [app](http://itunes.com/apps/flipclock) that uses it and it's getting a lot of downloads, which is nice. I also had a look at iAd, which I'll reserve judgement on for the time being, at the minute it's not looking particularly lucrative...

As well as iOS stuff, I also took a look at OpenGL. I tend to do this every now and again, each time dipping in a little further and understanding a bit more. This time round it was looking at shaders (in OpenGL ES 2.0) and integrating with [Bullet physics](http://www.ianvoyce.com/index.php/2011/11/18/creating-a-physics-based-opengl-ios-app/). My learning was certainly helped by this excellent [Learning Modern 3D Graphics Programming](http://arcsynthesis.org/gltut/) tutorial.


### Kinect and F#


Also while at home I finally got around to setting-up by my Kinect to do a bit of F# hacking. In the end I didn't produce [very much](http://www.ianvoyce.com/index.php/2011/09/05/kinect-sdk-with-f/), but it was pretty good fun anyway.



## Bring on 2012


So, to sum it up: more of the same really, I guess. Although this year there's definitely been a bit more of a focus on UI development, in both WPF and iOS, which I've been enjoying. It does tend to feel a bit schizophrenic, switching between technologies and platforms at the end of the working day. It can be interesting to identify the common themes and significant differences in the two approaches to GUI development. 

This year I'm hoping to make my blogging a bit more regular, but who knows if I'll be able to stick to it... If there's anything you'd particularly like me to focus on, feel free to leave a comment.

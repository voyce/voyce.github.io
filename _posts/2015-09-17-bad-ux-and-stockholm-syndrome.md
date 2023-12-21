---
date: 2015-09-17 12:19:25+00:00
description: ''
featuredImage: ''
slug: /bad-ux-and-stockholm-syndrome
template: blog-post
title: Bad UX and Stockholm Syndrome
categories:
- Rant
- Usability
tags:
- UI
- Usability
- ux
---

When we're thinking about the impact of good and bad UX on the experience of using software there are a large group of people who are often overlooked. They're not the discretionary consumers, who may be put off completing a purchase by requiring an extra click or form field, but a those who are forced to use in-house developed software required to complete their day-to-day work.

These apps are often clunky, with awkward UX, interaction foibles and bugs. They've quite possibly been ""designed"" by programmers, rather than UX specialists. But people have no choice but to use them day-in and day-out. So how is this likely to affect their attitude to the application?

An interaction can be so bad, so awkward to execute, that it feels almost painful. Maybe not when performed in isolation, but when repeated over and over it can become mentally (and even physically) wearing. 

The application has them captive, and sets about bending them to its will.

But then a strange thing happens. The users start to think it's OK. 



> Maybe it doesn't feel _quite_ as bad as it used to.  
> ...  
> It's not really _so_ awkward that it works that way.  
> ...  
> In fact, it feels right to do it this way.  
> ...  
> Actually, why would it be done any other way!?  



Wait a second, what just happened? 

Despite the terrible treatment meted out, and partly due to the captor showing some small kindness - giving the ability to do their job, however awkwardly - the hostages have fallen in love with the application, their cruel, heartless captor.
<!-- more -->


#### Why does it happen?


In a word: adaptation. People are very good at adapting to their mental and physical environments. It's a fundamental evolutionary trait that's helped us survive for millennia. 

You live in the Tibet? You're going to get used to drinking [Yak butter tea](http://www.webexhibits.org/butter/countries-tibet.html).
You use a clunky user interface? You're going to get used to clicking 3 times in a field before you can type something in. 

Putting aside the physical aspect for a second, there are 2 types of psychological adaptation: assimilation and accommodation. 

People can easily assimilate things that match their existing mental models, but if there's a mismatch then the more difficult act of accommodation is required. This relates very directly to a couple of important UX factors: 




  * Have the system match the user's mental model, not the underlying technical implementation model.



  * Make the model consistent. Changes and inconsistencies can easily undermine the value of a good mental model.
   

The difficulty here can be getting buy-in for the additional development and design effort required to build a good, user-facing abstraction. Unfortunately the easiest option is often to directly expose the nuts-and-bolts of the implementation.

On the physical side, we risk developing "bad" muscle memory; people doing the same awkward series of interactions enough for it to become something that feels easy.



#### Why is it bad for us as developers/UXers?


Once people have adapted to their environment (however harsh it may be), it can take a significant effort to change. 

The inertia of an entrenched behaviour can be hard to overcome; you'll need to convince users that the treatment they've become used to is not the way things have to be. You're going to have to convince them that their effort to adjust to a different, better way of doing things will be worth it. 

This can be a hard sell, even if the new way different way is demonstrably better, and they may push back more than you'd expect.



#### Don't make it hard for yourself - or them


So what's the moral of the story? Try hard not to let people fall in love with a bad UX to begin with. 

If you're forced to deliver something nasty quickly and then iteratively improve, make sure the iteration cycle is quick enough that your users don't become accustomed to a broken way of doing things in the meantime.

Invest the time to understand the underlying business/process model that the system or application is representing. Talk to users to understand how things proceed in the real world and enable them to use their intuition about these existing processes in your application. It will make them feel warm and comfortable.

And leave behind the handcuffs...

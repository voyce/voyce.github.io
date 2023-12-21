---
date: 2011-05-27 22:55:46+00:00
description: ''
excerpt: Async is a powerful way of getting parallelism in your F# code, just be careful
  what you call within your Async workflows.
featuredImage: build/gatsby/www.ianvoyce.com/assets/2011-05-27-fsharp-async-plays-well-with-others_threads_raw-71x300.png
slug: /fsharp-async-plays-well-with-others
template: blog-post
title: 'F# Async: Plays well with others?'
categories:
- .NET
- F#
tags:
- .NET
- async
- F#
- parallel
- threading
---

OK, quick [Async](http://msdn.microsoft.com/en-us/library/dd233250.aspx) pop quiz: How long does the `run` function below take to execute?

    
    
    module Test = 
        let work i = 
            async { 
                System.Threading.Thread.Sleep(500) 
                return i+1 
            }
        let run _ = 
            [1..1000] |> List.map work 
            |> Async.Parallel 
            |> Async.RunSynchronously
    


(Waits for people to start FSI and paste in the code...)

My guess would've been something just over 500ms; each of the 1000 async tasks would surely sleep in parallel, and then the operation itself is trivial. The additional elapsed time would be dominated by the overhead of thread management, and depend on the number of threads that can physically run in parallel (I'm using an 8-core machine). But still, something close to 500ms...

The actual result? 28000ms. Yes, you read that right: 28 seconds. What on earth did we do wrong? 
<!-- more -->
Surely it can't be that [Sleep](http://msdn.microsoft.com/en-us/library/d00bd51t.aspx), right? I mean, each task is going to run on the thread pool anyway, so what difference would that make? Well, that *is* exactly the problem. It turns out that you can't mix "vanilla" threading calls with `Async` API ones, because Async works in a completely different and incompatible way. If we use raw threading functions like `Sleep`, the async framework has no way of knowing if we're doing anything useful. It eventually ends up causing .NET thread pool starvation, as we're essentially grabbing and holding each thread in a way that doesn't cooperate with anything else. New threads will spin-up to compensate and we'll end up getting additional memory usage as well as worse performance. 

[caption id="attachment_1232" align="alignleft" width="71" caption="Nasty"][![Nasty](http://www.ianvoyce.com/wp-content/uploads/2011/05/threads_raw-71x300.png)](http://www.ianvoyce.com/wp-content/uploads/2011/05/threads_raw.png)[/caption] This is a very rough impression of how things look if you use non-async aware blocking. Vertical lines represent threads in the thread pool, artificially restricted to 3 for diagrammatic purposes. Notice how each of the 'blocks of work' are large, and the opportunities to spread the work amongst threads is small. As a result, total elapsed time is large.
  


Instead Async uses what's called [continuation passing style](http://en.wikipedia.org/wiki/Continuation-passing_style), a technique favoured by the functional programming community, that also happens to map quite nicely to features in the CLR, including the thread pool, tail calls and garbage collection. It has the rather nice feature that it never blocks an OS thread (unless it's busy actually executing). Instead it queues a function (a continuation) to be called at some later point. 

[caption id="attachment_1233" align="alignright" width="183" caption="Nice"][![Nice](http://www.ianvoyce.com/wp-content/uploads/2011/05/threads_async-183x300.png)](http://www.ianvoyce.com/wp-content/uploads/2011/05/threads_async.png)[/caption]Here the blocks are broken into a small amount of initial work to create and schedule the event, and then a further small amount to execute the continuation. In the meantime other code can be run (in this case, other instances of the same task), and we get good usage of the thread pool.  


For all the gory details and full Async semantics, I suggest you read the [paper](http://research.microsoft.com/pubs/147194/async-padl-revised-v2.pdf) from Microsoft Research. Tucked away in there you'll see that they do mention this restriction: 



<blockquote>The key facet of an asynchronous I/O primitive is that it does not block an OS thread while executing, but instead schedules the continuation of the asynchronous computation as a callback in response to an event.
</blockquote>



So how do we "fix" our example? Simple: we replace the non-CPS-friendly explicit Sleep, with an Async-native Sleep.  


    
    
    module Test = 
        let work i = 
            async { 
                do! Async.Sleep(500) 
                return i+1 
            }
        let run _ = 
            [1..1000] |> List.map work 
            |> Async.Parallel 
            |> Async.RunSynchronously
    



The result now: 558ms. Much more like it! 

What happens is that the `Async.Sleep` call is converted into an event that is fired after 500ms to continue the execution, i.e. actually perform the addition. In the meantime, each of the other instances of the workflow get a chance to run, and we get real parallelism.   

In this case it might look screamingly obvious, but be aware if you're doing something such as calling into a third-party, legacy, or otherwise non-Async-aware code, it could cause your otherwise highly parallel code to choke. You'll have to wrap it up in order to make it play nicely with Async and make sure you get the full benefit of this powerful F# feature.

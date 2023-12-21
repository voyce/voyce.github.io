---
date: 2007-04-20 22:41:31+00:00
description: ''
featuredImage: ''
slug: /plinq-parallel-linq
template: blog-post
title: PLINQ - Parallel LINQ
categories:
- .NET
- Software Development
---

One of the most interesting things in terms of performance that comes out of the move to a more declarative style when using LINQ, is the ability to easily parallelise your computations.


This benefit has been widely understood for some time in the functional programming world, where side-effect free functions can be exploited in combination with higher order functions like map and reduce to process data in parallel. Joel Spolsky gives a good background to that [here](http://www.joelonsoftware.com/items/2006/08/01.html), in the the context of the Google [MapReduce ](http://labs.google.com/papers/mapreduce.html)framework.

Well, now it looks like Microsoft is finally catching up with the parallel data processing meme. CLR and concurrency expert Joe Duffy is lead developer on the parallel LINQ (PLINQ) project, that hopes to be able to automatically parallelise LINQ queries across multiple cores or processors. Joe’s excellent technical blog contains some tidbits of PLINQ information (as well as a lot of detailed concurrency-oriented posts) including a link to his [presentation](http://www.bluebytesoftware.com/code/07/01/2007_01_POPL_DAMP_plinq.ppt) from the Decalarative Aspects of Multi-core Processing conference, that includes both a high-level overview and some interesting implementation details.

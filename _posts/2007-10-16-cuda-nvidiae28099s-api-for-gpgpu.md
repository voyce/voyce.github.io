---
date: 2007-10-16 22:33:25+00:00
description: ''
featuredImage: ''
slug: /cuda-nvidia%e2%80%99s-api-for-gpgpu
template: blog-post
title: CUDA - nVidia’s API for GPGPU
categories:
- Software Development
---

Today I was at the Oxford eResearch Center for an nVidia [CUDA](http://developer.nvidia.com/object/cuda.html) workshop. CUDA is their new API for facilitating scientific computing on GPU (graphics processing units).


For a while now there’s been some interest within the computer science research community on the possible application of GPUs to general purpose computing tasks. Well, it’s more accurate to call them scientific computing tasks, I guess, as these are typically embarassingly parallel numerical applications like monte carlo simulations.

nVidia are targeting CUDA directly at potential users in the financial, oil & gas and aerospace industries. These are areas where they’ve never had a significant presence, instead being focused on the consumer gaming and graphics workstation market, which has served them very well. But now they’re looking to exploit the diminishing gains in processor power in recent generations of processors by instead helping people to squeeze maximum bang-for-the-buck by utilising the highly-threaded, parallel, multi-processor nature of commodity GPUs.

The presentations were very interesting. They ranged from almost marketing level discussions of the product road maps etc, to real nuts-and-bolts optimisation discussions. Mark Harris demonstrated successive iterations of a reduction algorithm that eventually approached the limit of memory bandwidth (78gb/s), albeit with some pretty nasty loop unrolling and templating!

From my perspective the biggest drawback of the current hardware and API is that it only supports single precision floating point. Unfortunately everything in my world uses double precision maths; and although it’s possible to convert on entry/exit from the GPU API, this adds significant overhead. Of course, it should be possible to reduce the numerical range that the algorithm has to deal with in order to avoid the need for double precision at all, but this is a bit more re-engineering than I can justify. Even the on-chip double implementation will suffer from a quite significant slow-down compared to the single precision version - but even this will be an order of magnitude faster than non-GPU code, so this shouldn’t be too serious.

Another interesting aspect of the technology is the use of an “intermediate” form of assembled code; PTX files. These are generated from the C-like .CU source files, and are then turned into card-specific machine code on the hardware. This allows nVidia a degree of freedom to change the on-chip architecture, instruction set etc without breaking existing applications.

If you’re interested in keeping in touch with news of the various GPGPU users in academia and industry, you can take a look at [www.gpgpu.org](http://www.gpgpu.org/), which was started by [Mark Harris](http://www.markmark.net/), who was one of the presenters today.

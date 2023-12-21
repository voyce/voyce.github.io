---
date: 2009-01-05 23:10:48+00:00
description: ''
featuredImage: ''
slug: /sound-formats-for-iphone-development
template: blog-post
title: Sound formats for iPhone development
categories:
- iPhone
- Mac
- Software Development
tags:
- caf
- endian
- ffmpeg
- iPhone
- libsndfile
- Mac
- wav
---

Ahh, back to work today. It's pretty tough getting back into the swing of things after what turned out to be a long break this year. While I was off I finally got to spend some time working on an iPhone game. After getting hold of the SDK a while back, it's only now that I've gotten around to doing something with it.

One of the things that seemed a little odd about the SDK is it's use of CAF-format audio files, detailed [here](http://developer.apple.com/documentation/MusicAudio/Reference/CAFSpec/CAF_intro/chapter_1_section_1.html). I got hold of a few very nice audio samples from the [freesound](http://www.freesound.org) site, but needed to convert them from WAVs to CAFs.

I thought ffmpeg might be up to the job, but the version I had didn't list it as an avaliable output format using `ffmpeg -formats`. However after a bit of digging I discovered that it is supported by [libsndfile](http://www.mega-nerd.com/libsndfile/), so set about installing it using [MacPorts](http://www.macports.org):


<blockquote>`sudo port install libsndfile`</blockquote>


Then I used the included libsndfile-convert app to convert my file:


<blockquote>`libsndfile-convert file.wav file.caf`</blockquote>


The output format is inferred from the file extension, so you don't have to specify it. However, when I rebuilt and ran my iPhone app using the new file, it didn't play back. I suspected there may be something wrong with the format of the file, so I took a look to see what [file](http://en.wikipedia.org/wiki/File_(Unix)) reported. For the original WAV file I got


<blockquote>`file.wav: RIFF **(little-endian)** data, WAVE audio, Microsoft PCM, 16 bit, mono 44100 Hz`</blockquote>


Unfortunately file doesn't work on .CAF files, but you can open them using QuickTime Player, and using the Movie Inspector window you can see that the file has the following format:


<blockquote>`16-bit Integer **(Big Endian)**, Mono, 44.100 kHz`</blockquote>


So it looks like the problem may be libsndfile-convert changing the endian-ness of the file contents, from the x86-style little-endian to Motorola-ish (i.e. pre-x86 Mac) big-endian, which is a bit of a pain. According to the docs, the libsndfile API supports endian-ness manipulation, so it's probably just the case that the helper app is doing the wrong thing automatically. I'll look at putting together a small command line app to use the API directly and enable me to batch process .WAV files correctly.

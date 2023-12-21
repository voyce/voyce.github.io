---
date: 2009-01-19 13:41:46+00:00
description: ''
featuredImage: ''
slug: /f-ctp-and-visual-studio-integration
template: blog-post
title: F# CTP and Visual Studio integration
categories:
- F#
tags:
- ctp
- F#
- targetpath
- visualstudio
- vs
---

Just a quick note on an inconsistency in the F# 1.9.6.2 (CTP) release and it's integration into Visual Studio: be aware that the standard VS environment variable `$(TargetPath)` is not getting set to what you'd expect. Rather than containing the full path to the output file it references the intermediate file typically in \obj\bin.

This can be a problem if you've got any tools set-up that try and do something with the built binary. Normally you can assume that referenced assemblies will also be in that directory, so you'd be able to load and execute your built file. If you're pointing to a copy in the intermediate directory, that's not the case.

It looks like it's just an artifact of the way they've integrated the F# compiler (fsc.exe) with msbuild. The F# team are aware of this bug, so hopefully it'll be fixed in the next drop.

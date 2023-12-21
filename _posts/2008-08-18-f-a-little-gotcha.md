---
date: 2008-08-18 13:20:43+00:00
description: ''
featuredImage: ''
slug: /f-a-little-gotcha
template: blog-post
title: F# - A little gotcha with GuidAttribute
categories:
- .NET
- COM
- F#
tags:
- COM
- F#
- guidattribute fsharp
---

Be careful when using the [<Guid("...")>] attribute on your COM-visible classes in F#. If you mistakenly use the curly-bracket delimited format for the GUID, regasm will silently, yes, _silently_, fail to add any CLSID entries for your class. That means it will be cocreatable by the prog ID, but not the CLSID. Ouch.

No doubt this will be addressed in the CTP release, due in a couple of weeks.

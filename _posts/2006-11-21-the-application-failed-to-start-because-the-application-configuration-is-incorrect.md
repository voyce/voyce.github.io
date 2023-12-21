---
date: 2006-11-21 22:52:59+00:00
description: ''
featuredImage: ''
slug: /the-application-failed-to-start-because-the-application-configuration-is-incorrect
template: blog-post
title: The application failed to start because the application configuration is incorrect
categories:
- Windows
---

Have you been seeing “The application failed to start because the application configuration is incorrect” errors? I’ve been doing some work on Windows Side-by-Side (SxS) stuff recently, and this is par for the course. One thing that not many people seem to know is that you should be able to get a lot more information on why this error is occurring by looking in the event viewer.

Either do Start|Run eventvwr, or go to Start|Control Panel|Administrative Tools|Event Viewer. Look in the System section for entries where the source is SideBySide. You should see a couple of errors generated for each failure.

The first one is generally not much use, but the second one contains the error that caused the failure. This is often:



	
  * A syntax error in the application configuration file (myapp.exe.config)

	
  * A dependent “assembly” specified in the manifest is missing


The config error is easier to fix, as these are external files that can be edited with any text editor. Manifest errors are more difficult as they can also be embedded in the binary as a resource.

---
date: 2014-01-27 22:50:48+00:00
description: ''
featuredImage: ''
slug: /ios-opengl-es-compatibility-gotcha
template: blog-post
title: iOS OpenGL ES compatibility gotcha
categories:
- Debugging
- iOS
- Software Development
tags:
- EAGLContext
- iOS
- OpenGL
- OpenGL ES
---

Oh man, I just wasted too many hours of my life trying to figure out why calls to [glBlitFramebuffer](http://www.khronos.org/opengles/sdk/docs/man3/xhtml/glBlitFramebuffer.xml) in my iOS app were returning GL_INVALID_OPERATION. I was targeting iOS 7, so I should've been able to use OpenGL ES 3.0 calls, and after all, I'd built against the v3 headers and everything else was compiling and working... right? 

Wrong. Well, I really should've RTFM. It turns out that ES 3.0 use is not determined just by the OS version, but by the hardware. So even if you're running iOS 7, you can only use ES 3.0 if you're on the latest gen: iPhone 5S, iPad Air etc. Check out the full compatibility matrix [here](https://developer.apple.com/library/ios/documentation/DeviceInformation/Reference/iOSDeviceCompatibility/OpenGLESPlatforms/OpenGLESPlatforms.html).

Here are a few extra tips to help you avoid wasting your time like I did if you're explicitly targeting OpenGL ES 3.0:



	
  * As well as running on the right hardware, you can also use the simulator, which supports emulation of v3.0.


  * Call glGetString(GL_VERSION) to get a report of which version you're actually running.


  * Pass the appropriate parameter to EAGLContext initWithAPI, if you're using it.
e.g.

    
    
     self.context = [[EAGLContext alloc] initWithAPI:
        kEAGLRenderingAPIOpenGLES3];
    






It's pretty frustrating that you get no indication that the function's not supported, as opposed to just having being passed bad state. But I guess that's par-for-the-course with a bare-bones, down-to-the-metal API like Open GL.

Sigh.

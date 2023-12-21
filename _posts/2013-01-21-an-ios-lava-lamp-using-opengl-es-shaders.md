---
date: 2013-01-21 23:56:55+00:00
description: ''
featuredImage: build/gatsby/www.ianvoyce.com/assets/2013-01-21-an-ios-lava-lamp-using-opengl-es-shaders_lavalamp-200x300.png
slug: /an-ios-lava-lamp-using-opengl-es-shaders
template: blog-post
title: An iOS Lava Lamp using OpenGL ES shaders
categories:
- C++
- iOS
- iPhone
- Mac
- Software Development
---

![Screenshot of the finished lava lamp effect](/assets/2013-01-21-an-ios-lava-lamp-using-opengl-es-shaders_lavalamp-200x300.png){: align="left"}
Catchy title, eh? This little experiment came about as I've been working on an iOS app where I decided to use an embedded OpenGL view, via GLKit, for a bit more flexibility than a plain-old UIView. This found me falling head-first down a rabbit-hole of OpenGL ES shaders. I ended up putting together a little demo that emulates a lava lamp using a nifty bit of GLSL code.
<!-- more -->


### A bit of background - shaders


OpenGL ES 2.0 introduced the idea of programmable shaders to replace the 'fixed function pipeline' of previous versions. What does this mean? You can write little programs that are invoked to modify vertices (points in the model) or fragments (pixels on the screen). Other types of shaders have been introduced in other versions of Open GL for desktop platforms, but vertex and fragment shaders are your limit on iOS/Open GL ES 2.0. 

The shader programs are written in a C-like language, and then compiled (at app run-time) into code that can then be run directly by the GPU. That means they get executed on the highly data-parallel graphics hardware, running the same code on multiple lightweight 'threads' (not your run-of-the-mill OS threads).

It's possible to write shader programs that get quite complex. You can define and call your own functions, use common functions - for helping with things like vector manipulation and linear interpolation - and make use of built-in OpenGL variables to read and write key data. Here's an example of getting the length of a 2D vector, the straight-line distance between 2 points.
```
float dist = length(vec2(balls[i].xy - p));
```   
That code also demonstrates the use of [swizzling](http://en.wikipedia.org/wiki/Swizzling_(computer_graphics)); a concise way of accessing and mapping elements from one matrix or vector to another. Specifically `balls[i]` is a 3D vector with x, y and z components and we're creating a 2D vector from only x and y. 



### Make a quad and paint it


In the case of our app, we're not doing much in the vertex shader, we're just going to pass vertices through unchanged. All we need to do is draw a single, screen-aligned quad (consisting of 2 triangles), and then "paint" it using our fragment shader. [caption id="attachment_1607" align="alignleft" width="234"][![Vertices](http://www.ianvoyce.com/wp-content/uploads/2013/01/vertex-diagram-234x300.png)](http://www.ianvoyce.com/wp-content/uploads/2013/01/vertex-diagram.png) Drawing a quad with a triangle strip [/caption]

By using an orthographic projection we can remove the need to take any perspective adjustments into account and just use the screen width and height for the size of the quad. Using a combination of UIView and [GLKit](http://developer.apple.com/library/mac/#documentation/GLkit/Reference/GLKit_Collection/_index.html), we can easily create the project matrix we need:

```
    float w = self.view.bounds.size.width;  
    float h = self.view.bounds.size.height;
    _projectionMatrix = GLKMatrix4MakeOrtho(-(w/2), w/2,
                                            -(h/2),
                                            h/2,  
                                            0.0, 1.0);
```

I also set the [contentScaleFactor on UIView](http://developer.apple.com/library/ios/#documentation/uikit/reference/uiview_class/uiview/uiview.html) to 1.0 - it defaults to 2.0 on Retina devices - in order to reduce the actual number of pixels that get drawn.



### The meat: The metaballs


So now that we've got our quad on the screen, what are we actually going to draw into it? There are lots of metaball tutorials and descriptions around, [this](http://www.niksula.hut.fi/~hkankaan/Homepages/metaballs.html) is a very useful one. With a little inspiration from [this stackoverflow answer](http://stackoverflow.com/a/3587981/1719), I got it coded up in GLSL:

Here I'm also setting a graduated colour in the fragment shader by returning the level rather then simply a boolean "in or out" flag. This gives a pleasing 'glow' to the blobs.

[caption id="attachment_1627" align="alignright" width="210"][![Metaballs formula](http://www.ianvoyce.com/wp-content/uploads/2013/01/metaballs_formula.png)](http://www.ianvoyce.com/wp-content/uploads/2013/01/metaballs_formula.png) Calculating the pixel value with one metaball[/caption]The crux of the algorithm involves dividing the radius of the blob by its distance from the point being rendered, as we can see in this diagram. We do the same thing for each blob, and the aggregate gives us the resulting colour for each pixel.    

This isn't a particularly efficient implementation. It gives me a solid 30 FPS on an iPhone 4S or iPad 2 with 5 balls, but increasing that number quickly starts to impact the performance. I'm sure there are some optimisations that could be made (e.g. marching cubes), but that wasn't really what this experiment was about. 



### Debugging


XCode provides excellent support for debugging OpenGL apps; albeit only on an actual device, not the the simulator. While your app is running, look for the somewhat hidden button or select Product\|Debug\|Capture OpenGL ES Frame, it will grab the state of your app and let you explore all of your state at will, including the very powerful ability to edit and recompile shaders on the fly. Very nice. 

You can also use View\|Show Debug Navigator to see the frame rate as your app is running. There's a good introduction to what you can do in the official Apple docs [here](http://developer.apple.com/library/ios/#recipes/xcode_help-debugger/articles/debugging_opengl_es_frame.html).



### The Result



It looks pretty nice, I think. I added some code to manage the balls, giving them an x, y and z (radius) direction so that they move and grow, as well as a bit of adjustment when they reach the top and bottom to make things look a little more realistic. I also added a small adjustment to the colours across the width of the display, to provide an interesting visual effect (it was originally intended to give a sheen, as if the blobs where behind cylindrical glass, but this was how it ended up instead)!

So, I'm sure most of you are just going 'yeah, yeah, great description, but SHOW ME THE CODE', so [here it is](https://github.com/voyce/LavaLamp). Enjoy.

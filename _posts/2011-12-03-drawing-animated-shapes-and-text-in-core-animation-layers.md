---
date: 2011-12-03 12:30:01+00:00
description: ''
excerpt: Using paths and text to simple create composite animated graphics with Core
  Graphics, UIKit and Core Animation.
featuredImage: build/gatsby/www.ianvoyce.com/assets/2011-12-03-drawing-animated-shapes-and-text-in-core-animation-layers_star_hello.png
slug: /drawing-animated-shapes-and-text-in-core-animation-layers
template: blog-post
title: Drawing animated shapes and text in Core Animation layers
categories:
- iOS
- iPhone
- Mac
- Software Development
tags:
- CALayer
- Core Animation
- Core Graphics
- Graphics
- iOS
- iPad
- iPhone
- objective-c
---

[caption id="attachment_1399" align="alignleft" width="138" caption="Star and text"][![Star and text](build/gatsby/www.ianvoyce.com/assets/2011-12-03-drawing-animated-shapes-and-text-in-core-animation-layers_star_hello.png)](build/gatsby/www.ianvoyce.com/assets/2011-12-03-drawing-animated-shapes-and-text-in-core-animation-layers_star_hello.png)[/caption]The other day I was overcome by the desire to create an animated start-burst, price-tag type graphic with iOS. Time to break out some Core Graphics and Core Animation code. On the way to getting it going, I came across some interesting gotchas, which I thought it'd be useful to talk about here.
<!-- more -->


### Drawing the basic shape


The first step is to draw the 'star' shape. I decided to create a re-usable path, although at the minute I'm actually only drawing it once, so the benefits are limited. Doing this involves using the `CGPath...` versions of the Core Graphics functions, that add themselves to a mutable path objects, rather than being drawn directly on the currently active graphics context.

    
    
        CGMutablePathRef r = CGPathCreateMutable();
        // Always have to move to a start point
        CGPathMoveToPoint(r, NULL, x1, y1); 
        CGPathAddLineToPoint(r, NULL, x2, y2);
        CGPathCloseSubpath(r);
    


Once you've got created the path you can merrily set various stroke and fill colours and draw it over-and-over to your hearts content.
 
[![star_algo](http://www.ianvoyce.com/wp-content/uploads/2011/12/star_algo.png)](http://www.ianvoyce.com/wp-content/uploads/2011/12/star_algo.png)The star drawing is parameterised in a few ways: we can set the inner radius (r2), outer radius (r1) and the number of points, which must be divisible by 3. We draw the points in groups of three, the first on the outer radius, then the inner, then the last on the outer. In that way the sections tesselate into a complete circle.


### Using it with CALayer


So now we've got some code to draw the shape we want, we need a way to get it onto the screen, using `CALayer`s. There are 3 main ways of providing content for layers:




  * Use a delegate that implements `drawInContext` (and don't forget to call `setNeedsDisplay` at least once to cause it be drawn!)


  * Set the content to a `CGImageRef`. Meh... that's going to mean it's a bitmap, with all the associated aliasing/scaling issues.


  * Subclassing. Nah.


We'll use the first approach; we can create a simple `NSObject`-derived class that can manage the layer hierarchy (more of that later) and implement the required selector on it. In that function we can switch on the layer we're being asked to draw, and do the appropriate handling:

    
    
    - (void)drawLayer:(CALayer *)theLayer
            inContext:(CGContextRef)context 
    {
        if (theLayer == _textLayer) {
            // ...
        } else if (theLayer == _starLayer) {
            // ...
        }
    }
    


 

### Animating


We're only going to be doing fairly basic rotation animation here, so we can use `CABasicAnimation`. We use the key-value coding support to specify the `transform.rotation` property as the target. This is an alias for rotation around the Z axis, which is pointing "out" of the screen. We rotate from 0 to 2*pi radians, repeating essentially indefinitely by specifying a large repeatCount.

    
    
        CABasicAnimation *animation = 
            [CABasicAnimation animationWithKeyPath:@"transform.rotation"];
        animation.duration=8.0;
        animation.repeatCount=HUGE_VALF;
        animation.autoreverses=NO;
        animation.fromValue=[NSNumber numberWithFloat:0.0];
        animation.toValue=[NSNumber numberWithFloat:TWOPI];
        [_starLayer addAnimation:animation forKey:@"rotation"];
    




### Drawing the text


This was an interesting one. I originally started looking at a `CATextLayer`-based solution, but was surprised to find that it's quite difficult to get vertical alignment within a rectangle. Instead, I decided to use the `NSString` UIKit additions that provide enough drawing and - importantly - measuring functions for us to work out exactly where we need to place the text.    

One important thing to note here is that we're potentially mixing Core Graphics and UIKit functions here. They have different expectations about how to get hold of the required graphics context; with Core Graphics it's always passed explicitly, whereas UIKit will grab the current context. This means that if you try and call `drawInRect` within your `drawLayer` function, you'll see errors like "Invalid context: 0x0" on the console, and no output.

The solution is simple when you know how: tell UIKit about your explicit context, like this:

    
    
    - (void)drawLayer:(CALayer *)theLayer
            inContext:(CGContextRef)context 
    {
        // ...
        // Let UIKit know about this context
        UIGraphicsPushContext(context);
        // Because this function uses it internally...
        [myString drawInRect:r 
                         withFont:font
                lineBreakMode:UILineBreakModeClip 
                        alignment:UITextAlignmentCenter];
        UIGraphicsPopContext();
    }
    


By measuring the text before we draw, we can align it centrally vertically and fill the space horizontally, letting iOS worry about the horizontal alignment. `sizeWithFont` takes into account a bounding rectangle and our desired breaking/clipping options:

    
    
            CGSize sz = [s sizeWithFont:font 
                     constrainedToSize:theLayer.bounds.size 
                         lineBreakMode:UILineBreakModeClip];
    




### Setting up a layer hierarchy


Given that I wanted some parts of the thing to rotate, and others to be static, I needed to create multiple layers and put them together. This is very easily achieved by adding layers to the `subLayers` collection of our root layer, then we return the root layer and add that to the view.

[caption id="attachment_1391" align="alignright" width="160" caption="Layer arrangement"][![Layer arrangement](http://www.ianvoyce.com/wp-content/uploads/2011/12/star_layers1.png)](http://www.ianvoyce.com/wp-content/uploads/2011/12/star_layers1.png)[/caption]The layer set-up looks like this, with the root being empty, and having first the star layer, then the text layer added to the sublayers. This is just because `addSublayer` appends the sublayer, instead we could use the `insertSublayer` overloads to be explicit about the ordering we desire.  
  

The set-up function returns the root layer, and then we add that to our view:

    
    
        _star = [[StarLayer alloc] initWithRect:CGRectMake(100, 100, 100, 100)];
        [[self.view.layer] addSublayer:_star.root];   
    




### Next Steps


Here's a red star and random text sitting a bit incongruously in a prototype of a spelling app I'm writing. 



It would be good to create a whole load of stars (probably not with text in) and shoot them across the screen in a star-burst by generating a random direction/speed vector and animating their speeds, opacity and scale to make them fade out.

You can check out the code [here](https://github.com/voyce/StarLayer).

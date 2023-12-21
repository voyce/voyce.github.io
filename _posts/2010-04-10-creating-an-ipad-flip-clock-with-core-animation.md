---
date: 2010-04-10 17:48:04+00:00
description: ''
excerpt: Using Core Animation to create flip-card number animations on the iPad and
  iPhone
featuredImage: build/gatsby/www.ianvoyce.com/assets/2010-04-10-creating-an-ipad-flip-clock-with-core-animation_flipclock_one.png
slug: /creating-an-ipad-flip-clock-with-core-animation
template: blog-post
title: Creating an iPad flip-clock with Core Animation
categories:
- Graphics
- iPhone
- Mac
tags:
- animation
- Graphics
- iPad
- iPhone
---

[![flipclock_one](build/gatsby/www.ianvoyce.com/assets/2010-04-10-creating-an-ipad-flip-clock-with-core-animation_flipclock_one.png)](build/gatsby/www.ianvoyce.com/assets/2010-04-10-creating-an-ipad-flip-clock-with-core-animation_flipclock_one.png)As part of the sliding puzzle game I'm developing for the iPhone and iPad (well, I can't survive on the profits from [BattleFingers](http://voyce.com/BattleFingers/) forever), I looked for a way to represent a numeric score and time display in an interesting way. One of the nicer visual effects you could use for this is the "flip-card clock" style, where each number consists of a top and bottom part, and the top part flips down to reveal the next number. It's been used in a few other places including the home screen in the HTC Diamond device, and its physical, realistic style fits well with the iPad, so I set about creating a version for the iPhone and iPad using the built-in Core Animation library. Read on for more details.
<!-- more -->
I started by thinking about the motions that would be required for the animation. We can break it down as three elements, the top, the bottom and the flipping part. Each of these parts can be represented as a Core Animation Layer (a `CALayer`). We create them using the class method on CALayer, there's no additional `retain` here, as the `_toplayer` property is retained.

    
    
    _toplayer = [CALayer layer];
    




## Splitting images


Now we've got the layers, we need something to but into them. I didn't want to create lots of pre-processed images for the top and bottom sections of each number, so instead I wrote a routine to split the images in code, drawing each half into an image context and grabbing a `UIImage` from it. This is then stored in an array for later use. We can get any of the elements from the array and set it as the `content` of the layer as required.

    
    
    // The size of each part is half the height of the whole image
    CGSize size = CGSizeMake([img size].width, [img size].height/2);
    			
    // Create image-based graphics context for top half
    UIGraphicsBeginImageContext(size);
    			
    // Draw into context, bottom half is cropped off
    [img drawAtPoint:CGPointMake(0.0,0.0)];
    // Grab the current contents of the context as a UIImage 
    // and add it to our array
    UIImage *top = [UIGraphicsGetImageFromCurrentImageContext() retain];			
    [_imagesTop addObject:(id)top];
    
    // Now draw the image starting half way down, to get the bottom half
    [img drawAtPoint:CGPointMake(0.0,-[img size].height/2)];
    // And store that image in the array too
    UIImage *bottom = [UIGraphicsGetImageFromCurrentImageContext() retain];			
    [_imagesBottom addObject:(id)bottom];
     
    UIGraphicsEndImageContext();
    


Now our `_imagesTop` and `_imagesBottom` arrays contains all the images we need, let's get 'em moving.


## Applying animation


[caption id="attachment_808" align="alignright" width="202" caption="The axes and anchor point of a CALayer"][![The axes and anchor point of a CALayer](build/gatsby/www.ianvoyce.com/assets/2010-04-10-creating-an-ipad-flip-clock-with-core-animation_flipclock_axes1-202x300.png)](http://www.ianvoyce.com/wp-content/uploads/2010/04/flipclock_axes1.png)[/caption]The animation we need to apply to the 'flipping' part is a rotation around the bottom of the layer, at the join between the two halves. The default location of the anchor point is the middle of the layer, which would mean the piece would rotate around the middle, but we can easily change it to instead be at the middle in the bottom (for our purposes it could be anywhere on the bottom, as we're only interested in x-axis rotation):

    
    
    _fliplayer.anchorPoint = CGPointMake(0.5, 1.0);
    


I use the `CATransform3DMakeRotation` function to create a transform around the X axis using the vector [1,0,0]. For the first part of the motion, we go from 0 radians to pi/2 radians (0 to 90 degrees), using an explicit `CABasicAnimation` object:  

    
    
    CABasicAnimation *topAnim = [CABasicAnimation animationWithKeyPath:@"transform"];
    topAnim.duration=0.25;
    topAnim.repeatCount=1;
    topAnim.fromValue= [NSValue valueWithCATransform3D:CATransform3DMakeRotation(0.0, 1, 0, 0)];
    float f = -M_PI/2;
    topAnim.toValue=[NSValue valueWithCATransform3D:CATransform3DMakeRotation(f, 1, 0, 0)];
    topAnim.delegate = self;
    topAnim.removedOnCompletion = FALSE;
    [_fliplayer addAnimation:topAnim forKey:[NSString stringWithUTF8String:k_TopAnimKey]];
    




## Drawing layer content


Unfortunately a CALayer doesn't have a "back", so creating the flip layer is not as easy as it could be. We need to add a step in the process to change the image displayed by the flip layer half way through its fall. We can do this in a variety of ways, I've chosen to set a delegate object to perform the drawing of the layer contents:

    
    
    [_fliplayer setDelegate:_layerHelper];
    


This means our delegate object (`_layerHelper`) has its `drawLayer` method called whenever the layer needs to display its contents. Be wary though, this may not be as often as you think! Core graphics aggressively caches the contents of the layer, and will only call your delegate when absolutely necessary. You can force it to happen by calling `setNeedsDisplay`, which invalidates the contents of the layer:

    
    
    [_fliplayer setNeedsDisplay];
    


I encapsulated this in the helper object itself, by providing a custom implementation of the `isTop` property setter that calls `setNeedsDisplay`. At least that way callers don't have to be aware of this requirement.

The `drawLayer` method itself simply draws one of two images into the context, depending on whether it's in 'top' mode. A slight further complication is that when drawing the bottom part, the image needs to be inverted to display upside down. This is achieved by applying a translation and scale to the context before drawing the image: 

    
    
    CGContextSaveGState(context);
    CGContextTranslateCTM(context, 0.0f, r.size.height);
    CGContextScaleCTM(context, 1.0f, -1.0f);
    		
    CGContextDrawImage(context, r, [_imgTop CGImage]);
    		
    CGContextRestoreGState(context);
    




## Avoiding implicit animation


I've tried to avoid using implicit animation here, instead using explicit construction of `CABasicAnimation` objects, because we need some control of when things happen, and the order in which they happen. Normally, it would be easier to just set some property of the view and let the OS manage animating it. Many properties have default transition that are applied whenever they're changed, so it takes some additional effort to make them change immediately.

In our example we change the contents of the top half and show the flip layer, but we need that to happen immediately so we disable actions within an explicit transaction that we then commit. This gives us the control we need, and avoids the default behaviour of the transition occurring when the message loop next runs.

    
    
    [CATransaction begin];
    [CATransaction setValue:(id)kCFBooleanTrue
    				forKey:kCATransactionDisableActions];
    _toplayer.contents = (id)[[_imagesTop objectAtIndex:[self getNextIndex]] CGImage];
    _fliplayer.hidden = NO;
    [CATransaction commit];
    




## Getting some perspective


Although the images I've used to illustrate this post are isometric, what we really want is a front-on perspective view, so that the falling piece seems to stick out. For this we have to set a `sublayerTransform` on the view's inherent layer, by setting an individual element on a transform:

    
    
    CATransform3D aTransform = CATransform3DIdentity;
    float zDistance = 1000;
    aTransform.m34 = 1.0 / -zDistance;	
    [self layer].sublayerTransform = aTransform;
    


This gives us a much more realistic look.


## Putting it together


Wow, so now we've got some appropriately sliced images, some layers and some animations. Let's put them together to get the final effect. 
[caption id="attachment_812" align="aligncenter" width="300" caption="The animation states (last is same as first)"][![The animation states (last is same as first)](build/gatsby/www.ianvoyce.com/assets/2010-04-10-creating-an-ipad-flip-clock-with-core-animation_flipclock-300x96.png)](http://www.ianvoyce.com/wp-content/uploads/2010/04/flipclock.png)[/caption]
I've used a very simple state machine that moves through 3 states; TopToMiddle, MiddleToBottom, ResetForNext. We can use the `animationDidStop` delegate as the state transition point. We don't have to bother determining exactly which animation finished, because we do the same thing (change state) regardless.

    
    
    - (void)animationDidStop:(CAAnimation *)oldAnimation finished:(BOOL)flag
    {
    	[self changeState];
    }
    






  * **TopToMiddle**: show flip layer, and start rotation through 90 degrees


  * **MiddleToBottom**: change image on flip layer, start rotation through further 90 degrees


  * **ResetForNext**: change displayed top and bottom layer contents and hide flip layer


Let's see how it looks in action:


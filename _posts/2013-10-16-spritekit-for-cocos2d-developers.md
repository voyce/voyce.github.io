---
date: 2013-10-16 21:59:11+00:00
description: ''
featuredImage: build/gatsby/www.ianvoyce.com/assets/2013-10-16-spritekit-for-cocos2d-developers_spritekit.png
slug: /spritekit-for-cocos2d-developers
template: blog-post
title: SpriteKit for Cocos2D developers
categories:
- Graphics
- iOS
- iPhone
tags:
- 2d
- cocos2d
- games
- Graphics
- iOS
- OpenGL
- SpriteKit
- sprites
---

![spritekit logo](/assets/2013-10-16-spritekit-for-cocos2d-developers_spritekit.png){: align="left"}
On my recent iOS puzzler [Wordz](http://appstore.com/opcode/Wordz), I decided not to reinvent the wheel, and instead use an off-the-shelf 2d game framework. I settled on Cocos2d. It makes it very easy to put together sprite-based games or apps by providing all the basic pieces like a scene graph, animations and integration with a couple of physics engines. It's built on OpenGL but, happily, hides all that away from you - unless you need it. 

No sooner had I released it, than Apple came out and announced a new framework for 2d games: SpriteKit. And it's remarkably similar to Cocos2d. Here I'll take a look at a few places where they differ, so you know what to look out for if you're considering migrating to SpriteKit.
<!-- more -->


## Set-up and Scenes


The first steps in getting your app up and running are also the ones where you're likely to notice SpriteKit's better integration with UIKit. Cocos2d requires you to create a CCGLView, which internally creates and wraps a native CAEAGLLayer, and then associate it with a CCDirector. Because the view is created internally, it's difficult to use a .xib or storyboard to create your UI, which is a shame. Here's the relevant part of the code the Cocos wizard adds to the application delegate:

    
    
     // Create an CCGLView of the correct format
     CCGLView *glView = [CCGLView viewWithFrame:[window_ bounds]
                                    ...  various buffer format options...
                                numberOfSamples:0];
     director_ = (CCDirectorIOS*) [CCDirector sharedDirector];
     // attach the OpenGL view to the director
     [director_ setView:glView];
    


SpriteKit on the other hand provides a UIKit-derived class, SKView, that you can create directly in a .xib. You don't need any custom code in your app delegate and your view controller can simply access the view and use it. And there's no SpriteKit equivalent of the CCDirector, so instead of using pushScene/popScene, you transition between scenes in the view directly. All sprites have a reference back to the scene in which they live, as you can see here:

    
    
     [self.scene.view presentScene:newScene 
                        transition:reveal];
    




## Interaction


This is another area where SpriteKit uses platform-specificity to its advantage. Cocos requires the use of the CCTouchDispatcher, responsible for dispatching touches to handlers registered like so:

    
    
     [_touchDispatcher addTargetedDelegate:node]
                                  priority:1
                           swallowsTouches:YES];
    


Then it's up to the node to implements the methods it requires from CCTouchOneByOneDelegate: one or more of ccTouchBegan, Moved and Ended. It can get a little fiddly dealing with handlers, for instance when displaying a "modal" sprite over another scene that handles touches, you have to ensure the model one swallows touches rather than allowing them to go to the scene behind.

With SpriteKit it's just a case of setting userInteractionEnabled to TRUE, then handling the standard UIResponder methods, touchesBegan etc. There's even a UITouch extension that provides a means of transforming the touch location to node space.



## Labels


Text labels in games are often created using bitmap fonts. In other words, they use their own fonts, rather than those installed on the system. This is a bit of historical artefact really; from back in the days where games implemented their own user interfaces and weren't running on top of an existing well-established UI framework. Because they couldn't count on there being any fonts available, they shipped their own. 

Cocos2D provides CCLabelBMFont for creating this type of label. As well as the text to display, you also provide a bitmap image and a .fnt file that describes the rectangular areas within the image that make-up each character.

Perhaps surprisingly, SpriteKit provides no equivalent of the bitmap label. Instead, they're like Cocos's CCLabelTTF: specified using the name of a font, e.g.:

    
    
     SKLabelNode *myLabel = [SKLabelNode labelNodeWithFontNamed:@"Chalkduster"];
    

 
Because you can bundle custom fonts with your iOS app you win back some of the flexibility of a bitmap font, but you still lose the ability to transform, rotate, scale, etc. each character individually, which could be quite useful for that retro 'GAME OVER' display.



## Custom drawing


There's one more big thing that's missing from SpriteKit: custom drawing. In Cocos2D it's possible to get 'down to the metal' and override the draw method, giving you the chance to add or alter the OpenGL drawing in any way you see fit. For instance, I wrote some code to use the standard Cocos2d shader with raw OpenGL calls to draw custom shapes during the draw cycle: 

    
    
    -(void)draw {
        // Let base class draw itself first
        [super draw]; 
        
        // Access Cocos2d shaders
        CCGLProgram *shader = [[CCShaderCache sharedShaderCache]
                               programForKey:kCCShader_Position_uColor];
        [shader use];
    
        // Raw OpenGL calls    
        glVertexAttribPointer(kCCVertexAttrib_Position, 3, GL_FLOAT, GL_FALSE,
                              sizeof(ccVertex3F), (void*)vertices);
        glDrawArrays(GL_TRIANGLE_STRIP, 0, _numVerts);
    


You can imagine why Apple don't allow this: it's in their interest to hide the actual implementation of the drawing in such a way that they can improve, change or optimise it without breaking 3rd party developer's code. For instance, they might decide to have SpriteKit use OpenGL ES 3.0 extensions, but only on iPhone 5S devices. The paranoid might also say that it gives Apple a way of locking developers in to a specific platform, rather than allowing them to use the platform-agnostic OpenGL and run it anywhere.

Hopefully that's a useful run down of a few of the differences between SpriteKit and Cocos2D. 

Now go and make a game!

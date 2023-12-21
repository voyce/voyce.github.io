---
date: 2011-11-18 22:40:50+00:00
description: ''
excerpt: If you know a bit of OpenGL you can integrate the powerful Bullet physics
  API into your iPad or iPhone app.
featuredImage: build/gatsby/www.ianvoyce.com/assets/2011-11-18-creating-a-physics-based-opengl-ios-app_puppet_grab.png
slug: /creating-a-physics-based-opengl-ios-app
template: blog-post
title: Creating a physics-based OpenGL iOS app
categories:
- C++
- Graphics
- iOS
- iPhone
- Software Development
tags:
- Bullet
- Graphics
- iOS
- iPad
- iPhone
- OpenGL
- physics
---

![puppet_grab](build/gatsby/www.ianvoyce.com/assets/2011-11-18-creating-a-physics-based-opengl-ios-app_puppet_grab.png)With the success of iOS games like Angry Birds and its flocks of imitators, there are lots of people looking at creating physics-based games, so I decided to try and create a simple demo using OpenGL ES and the Bullet physics engine.
<!-- more -->


## OpenGL


The 3D work I'd done previously on iOS was using OpenGL ES 1.1, so as part of doing this, I also needed to get up to speed with the changes in version 2.0. The move involves replacing the fixed function pipeline approach with the use of shaders crafted in "shader language", specifically [GLSL](http://www.opengl.org/documentation/glsl/). This approach is more flexible, but it means you take full responsibility for combining all the relevant matrices (model, projection etc), rather than letting OpenGL do it for you.

There are 2 distinct types of shader programs that operate on either vertices (vertex shaders) or pixels (fragment shaders). The steps required to create shader programs mirror the process of creating "normal" programs: they're loaded, compiled and linked. Once you've got a program loaded, you can initialise it by setting variables of various kinds: attribute, uniform or varying. These are used for vertex and colour data, constants and passing data between vertex and fragment shaders respectively. If you're interested in learning more, I suggest you check out some of the plethora of internet and dead-tree material on OpenGL. I'll only be doing pretty basic stuff here.   



## Bullet


I decided to try using [Bullet](http://bulletphysics.org/wordpress/) as the physics engine. It looks very powerful and is widely used in various games and [movies](http://bulletphysics.org/wordpress/?p=241); the only bad thing I can say about it is that decent documentation is a bit lacking. I found the best way to get help was googling for API names on the Bullet forum; not a very efficient way of doing things. Maybe I've just been spoilt by the copious Apple and MS docs (yeah, yeah). 


### Building it


Luckily Bullet comes with an Xcode project "in the box", so with a bit of `autoconf` magic, it was soon up and building the demo apps. Although there are loads of files in there, the core library itself isn't too large or complex. As such, I decided to create a separate static library project targeting the iOS simulator (i386) and devices (armv6/7).


### API basics


My intention is to use only a small subset of the Bullet API, encompassing rigid bodies and a couple of simple constraint types. For the purposes of the demo, I'm going to create the pretty standard set-up of a large static ground-box, along with a collection of more dynamic objects.  

The key part of joining up Bullet with your existing API is the [btMotionState](http://www.bulletphysics.com/Bullet/BulletFull/classbtMotionState.html) class. It enables the API to callback into your code with updated positions at each 'tick' of the physics engine, and you can then alter your geometry appropriately. I created a simple class that derives from `btMotionState`, implements the required pure virtual methods and contains a reference to my world object. Bullet calls the subclassed methods and we pass the transformations on to the object.

    
    
    class QuadMotionState : public btMotionState {
        Quad *_object;
        btTransform _transform;
    public:
        QuadMotionState(const btTransform &initialpos, Quad *node) {
            _object = node;
            _transform = initialpos;
        }
    	
        virtual ~QuadMotionState() {
        }
    	
        void setNode(Quad *node) {
            _object = node;
        }
    	
        virtual void getWorldTransform(btTransform &worldTrans) const {
            worldTrans = _transform;
        }
    	
        virtual void setWorldTransform(const btTransform &worldTrans) {
            // Pass the worldTrans to _object
        }
    };
    
    





### Debugging


Now, obviously _you_ will write the perfect code first time, and will have no need for any kind of diagnostics. But I am not that perfect; I need to see the difference between what I've told the API to do, and what it's actually doing. You can do this by creating an object implementing `btIDebugDraw` and passing it to the (slightly oddly named) setDebugDrawer function on your world:

    
    
    btIDebugDraw *debug = new DebugDrawHelper();
    m_dynamicsWorld->setDebugDrawer(debug);
    



The most important function on `btIDebugDraw` is `drawLine`. It will be called repeatedly to draw axes and bounding boxes in a variety of colours to indicate the object's state. We can use the `Shader` we created earlier to give us an easy way of generating these debug visuals. All we need is a very simple shader, so can declare it inline:

    
    
    program.loadShaders(
    		"attribute vec4 position;\n"
    		"uniform mat4 projection;\n"
    		"void main () {\n"
    		"gl_Position = position * projection;\n"
    		"}\n", 
    							
    		"uniform highp vec3 color;\n"
    		"void main () {\n"
    		"gl_FragColor = vec4(color.r, color.g, color.b, 0.0);\n"
    		"}");
    


Of course, anything more complex than this and you'll want to put it into a separate file, but this program allows us to pass in the start and end vertex and its colour to draw the line in our implementation of `DrawLine`:

    
    
    virtual void drawLine(const btVector3& from,const btVector3& to,const btVector3& color) {
            glUseProgram(program.getProgram());
            
            // Set the projection matrix
            GLint pu = glGetUniformLocation(program.getProgram(), "projection");
            glUniformMatrix4fv(pu, 1, 0, _projection.Pointer());
            
            // Set the colour of the line
            GLint puc = glGetUniformLocation(program.getProgram(), "color");
            vec3 v = vec3(color.getX(), color.getY(), color.getZ());
            glUniform3fv(puc, 1, v.Pointer());
            
            // Set the line vertices
            float tmp[ 6 ] = { from.getX(), from.getY(), from.getZ(),
                to.getX(), to.getY(), to.getZ() };
            glVertexAttribPointer(0, 3, GL_FLOAT, false, 0, &tmp[0]);
            glEnableVertexAttribArray(0);
            glDrawArrays( GL_LINES, 0, 2 );
    }
    


[caption id="attachment_1327" align="alignright" width="226" caption="Bullet debug output"][![Bullet debug output](http://www.ianvoyce.com/wp-content/uploads/2010/09/physics_debug_screenshot-226x300.png)](http://www.ianvoyce.com/wp-content/uploads/2010/09/physics_debug_screenshot.png)[/caption] You'll end up with a display showing your quads, along with their orientation and state (whether they're "active" in the simulation). It looks something like this:
  



### Populating the world


So in order to have something nice to look at, we're gonna need to put some 'things' into our world. And in order to make it act like the real world we'll need to give those things some physical properties.

I created a simple class to render a textured box. This OpenGL object then needs to be represented in the physics worlds by a rigid body object, which in turn contains a collision detection object that closely matches the dimensions of the original object. In fact, in the case of a box, the collision detection object matches exactly, otherwise you'll need to create an approximation using the more advanced classes. We use a `btBoxShape` to initialise our `btRigidBody` instance, along with setting its mass, inertia, and making a link to the state object that I mentioned above.

    
    
    btRigidBody *addShape(Quad *shape, btScalar mass = 1.0f, float restitution = 0.2f)
    {
    	// Set the initial transform based on the current location of the object
            btTransform transform;
    	transform.setIdentity();
    	transform.setOrigin(btVector3(shape->getX(), shape->getY(), 0.));
    		
    	btCollisionShape* cshape = 
                new btBoxShape(btVector3(btScalar(shape->getWidth()/2),
                                         btScalar(shape->getHeight()/2),
                                         btScalar(shape->getDepth()/2)));
            btVector3 localInertia(0,0,0);
    	if (mass != 0.0f)
                cshape->calculateLocalInertia(mass,localInertia);
    		
            QuadMotionState *state = new QuadMotionState(transform, shape);
    	btRigidBody::btRigidBodyConstructionInfo rbInfo(mass, state, cshape, localInertia);
    	btRigidBody* body = new btRigidBody(rbInfo);
    		
    	body->setDamping(0.85, 0.85);
    	body->setRestitution(restitution);
    		
            // Add the rigid body to the Bullet physics world
    	m_dynamicsWorld->addRigidBody(body);
            
            // Establish the link between our object and the physics object
    	shape->setPhysicsObject(body);
    	return body;
    }
    


To make things look a bit more polished I added some textures to the quads, along with very simple normal mapping to get some lighting effects. I also added the ability to rotate the view and zoom in and out by using `UIGestureRecognizer`s and changing the camera transform appropriately. The "finishing" touch (yeah, right), is the ability to tap to add a new quad.


### The results


Here's how the final thing looks. It's very basic, I've hardly touched the surface of the Bullet API - there are no constraints in play, for instance - but it's still kind of fun to play with. It's running fairly slowly here, what with it being in the simulator, and being recorded. On a physical iPad it's much snappier.
Let me know if you'd be interested to see some more coverage of some of Bullet's constraints and hinges features.

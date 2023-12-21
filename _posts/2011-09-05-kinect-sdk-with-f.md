---
date: 2011-09-05 14:34:47+00:00
description: ''
excerpt: My first quick look at using the Kinect SDK with F#.
featuredImage: build/gatsby/www.ianvoyce.com/assets/2011-09-05-kinect-sdk-with-f_kinect.png
slug: /kinect-sdk-with-f
template: blog-post
title: Kinect SDK with F#
categories:
- .NET
- F#
- Graphics
- Software Development
- WPF
tags:
- .NET
- F#
- Graphics
- kinect
- wpf
---

[caption id="attachment_1283" align="alignleft" width="200" caption="Just what do you think you\'re doing, Dave?"][![Just what do you think you're doing, Dave?](build/gatsby/www.ianvoyce.com/assets/2011-09-05-kinect-sdk-with-f_kinect.png)](build/gatsby/www.ianvoyce.com/assets/2011-09-05-kinect-sdk-with-f_kinect.png)[/caption]I finally got around to taking a look at the [Kinect SDK](http://research.microsoft.com/en-us/um/redmond/projects/kinectsdk/) the other day, partly because I was interested to see how the API looked from F#. Unfortunately getting it going turned out to be more of a pain than I was expecting.

The first bit was easy: I'm "lucky" enough to have one of the older Xboxes, which meant I'd had to get a Kinect with separate power, which is the one required by the SDK. Now all I needed was a Windows machine to develop on. 
<!-- more -->
For all my Visual Studio stuff at home I use a virtual machine, and unfortunately I missed the point in the [readme](http://research.microsoft.com/en-us/um/redmond/projects/kinectsdk/docs/readme.htm) about "Kinect for Windows applications cannot run in a virtual machine". Doh. That would explain why, try as I might I couldn't get [VirtualBox](http://www.virtualbox.org/) to detect the device when I plugged it into the host. Whatever I did I ended up with a 'resource is busy' error. 

I even tried another VM, this time from VMWare. It got further, with the guest seeing the devices, but whenever I tried to run the sample apps the API initialisation call failed. Unfortunately the samples make a bit of a rookie mistake and don't display the error code associated with the failure; the only way to get the underlying HRESULT is to debug the app. As it turned out the error was the catch-all `80080014`, which is attributed to various USB issues.

So, I finally relented and decided to once again set-up Bootcamp, which I'd stopped using a while back: why bother when VMs had done everything I needed? Again I was snookered, this time by the incompatibility of the [EFI](http://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) firmware in my ageing MacBook with the Win7 x64 install disc (which I'd had to actually burn onto optical media - can't remember the last time I burnt a real physical disk)!

Eventually, I got Win7 x86, Visual Studio and the Kinect SDK installed on the metal, plugged in the sensor and - whoa - the devices were recognised and the drivers installed and... the samples ran!

From this point doing some hacking was pretty straightforward. I set about creating a project that used the skeletal tracking ability from the SDK. The project needs to reference the `Microsoft.Research.Kinect` assembly, then we can open some namespaces and initialise the API:


    
    
    open System
    open Microsoft.Research.Kinect
    
    let nui = Nui.Runtime()
    nui.Initialize(Nui.RuntimeOptions.UseSkeletalTracking)
    



The API uses .NET events to communicate back to the application that some form of data is available. Depending on the options that you specify at init time, any of the skeletal, depth frame or colour data will be returned in the event arguments. This seems like a good place to use F# Async's event integration: Async.AwaitEvent. We can quite easily write some code that will create an Async task that will repeatedly listen for events:

    
    
        let skeleton (nui : Nui.Runtime) =
            let rec loop () =
                async { 
                    let! args = Async.AwaitEvent nui.SkeletonFrameReady
                    // Do something with the args
                    return! loop () 
                }
            loop ()
        // Start the process of listening for events...
        skeleton nui |> Async.Start
    


This seems to work nicely, although I'm not sure of the overhead involved, perhaps a solution involving Rx would be more appropriate. Anyway, let's do something with the data we get in the event, like getting all the skeletons that are being tracked, and passing them to a `draw` function:

    
    
                    args.SkeletonFrame.Skeletons 
                    |> Seq.filter (fun s -> s.TrackingState = Nui.SkeletonTrackingState.Tracked)
                    |> Seq.iter draw
    


All that's left is doing something cool in `draw`! Oh, and setting up all the GUI stuff necessary to actually get some pixels on the screen. I went for the WPF approach (as the managed samples do), which involves creating a simple object tree to display a rectangle in a canvas in a grid in a window, and doing a bit of marshalling back from the thread pool (where our Async code runs) to the main GUI thread.

Here's all the code for possibly the most tedious thing you could do with a Kinect! But hey, it's a starting point, right? I'm sure at some point I'll get around to doing something cooler than a couple of small red rectangles with it...


    
    
    open System
    open System.Windows
    open Microsoft.Research.Kinect
    
    [<stathread>]
    do
        let nui = Nui.Runtime()
        nui.Initialize(Nui.RuntimeOptions.UseSkeletalTracking)
    
        // lifted straight from the sample code
        let getDisplayPosition w h (joint : Nui.Joint) =
                let mutable depthX = 0.0f
                let mutable depthY = 0.0f
                nui.SkeletonEngine.SkeletonToDepthImage(joint.Position, &depthX, &depthY)
                let depthX = depthX * 320.0f; //convert to 320, 240 space
                let depthY = depthY * 240.0f; //convert to 320, 240 space
                let mutable colorX = 0
                let mutable colorY = 0
                let iv = new Nui.ImageViewArea()
                // only ImageResolution.Resolution640x480 is supported at this point
                nui.NuiCamera.GetColorPixelCoordinatesFromDepthPixel(Nui.ImageResolution.Resolution640x480, iv, 
                    (int)depthX, (int)depthY, 0s, &colorX, &colorY)
    
                // map back to visible area
                new Point((w * (float)colorX / 640.), (h * (float)colorY / 480.))
    
        // Set-up the WPF window and its contents
        let width = 1024.
        let height = 768.
        let w = Window(Width=width, Height=height)
        let g = Controls.Grid()
        let c = Controls.Canvas()
        let hd = Shapes.Rectangle(Fill=Media.Brushes.Red, Width=10., Height=10.)
        ignore <| c.Children.Add hd
        ignore <| g.Children.Add c
        w.Content <- g
        
        // We simple move the rectangle to where the head is
        let draw (s : Nui.SkeletonData) =
            let point = getDisplayPosition width height s.Joints.[Nui.JointID.Head]
            ignore <| w.Dispatcher.BeginInvoke(Threading.DispatcherPriority.Normal, Action(fun () ->    
                hd.SetValue(Controls.Canvas.TopProperty, point.Y)
                hd.SetValue(Controls.Canvas.LeftProperty, point.X)))
    
        let skeleton (nui : Nui.Runtime) =
            let rec loop () =
                async { 
                    let! args = Async.AwaitEvent nui.SkeletonFrameReady
                    args.SkeletonFrame.Skeletons 
                    |> Seq.filter (fun s -> s.TrackingState = Nui.SkeletonTrackingState.Tracked)
                    |> Seq.iter draw
                    return! loop () 
                }
            loop ()
    
        skeleton nui |> Async.Start
        
        let a = System.Windows.Application()
        ignore <| a.Run(w)
    
        nui.Uninitialize()
    


The moral of the story? Don't be like me, and make sure you actually read the FAQing readme, then maybe you'll spend more time doing a decent demo and less time plugging and unplugging your Kinect!

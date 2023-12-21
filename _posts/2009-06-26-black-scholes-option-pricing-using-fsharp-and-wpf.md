---
date: 2009-06-26 16:17:10+00:00
description: ''
excerpt: Using F# to create simple plots of Black-Scholes option prices and greeks
  using WPF.
featuredImage: build/gatsby/www.ianvoyce.com/assets/2009-06-26-black-scholes-option-pricing-using-fsharp-and-wpf_graph_call_expiry-150x150.png
slug: /black-scholes-option-pricing-using-fsharp-and-wpf
template: blog-post
title: Visualising Black-Scholes option pricing using F# and WPF
categories:
- .NET
- F#
- Software Development
tags:
- .NET
- F#
- wpf
---

The other day I was looking for an example of some code relevant to finance that I could use as a test for experimenting with Windows Presentation Foundation (WPF) and F#. I decided I could use a simple Black-Scholes option pricer combined with WPF to easily visualise how the option inputs affect the value and greeks. For example, given all the usual Black-Scholes assumptions, and a non-zero interest rate, you'd expect the value of an at-the-money call option to increase as time to expiry increases, whereas a put option would decrease in value.



### Calculating option values using F#


The first thing you need is an approximation to the cumulative normal distribution. I used the Abromowitz and Stegun approximation, which should give us enough precision. It's fairly concise to implement in F#:

    
    
    let ncdf x =
      let b1 =  0.319381530
      let b2 = -0.356563782
      let b3 =  1.781477937
      let b4 = -1.821255978
      let b5 =  1.330274429
      let p  =  0.2316419
      let c  =  0.39894228
      match x with
      | x when x >= 0.0 ->
        let t = 1.0 / (1.0 + p * x)
        (1.0 - c * Math.Exp( -x * x / 2.0)* t * (t*(t*(t*(t*b5+b4)+b3)+b2)+b1))
      | _ ->
        let t = 1.0 / (1.0 - p * x)
        (c * Math.Exp( -x * x / 2.0)* t * (t*(t*(t*(t*b5+b4)+b3)+b2)+b1))
    


The CDF should be near 0 for low values, around 0.5 for zero, and near 1 for high values. Let's check:

    
    
    > [-5.0; 0.0; 5.0] |> List.map ncdf;;
    val it : float list = [2.871049992e-07; 0.500000001; 0.9999997129]
    


Looks good.

Now let's put it to use by implementing the option pricing formula for call options:

    
    
    let call strike spot (rate:float) (now:float) (expiry:float) (vol:float) =
        let exp = expiry-now
        let d1 = (Math.Log(spot/strike) + ((rate+(vol*vol)/2.0)*exp))/(vol * Math.Sqrt(exp))
        let d2 = d1 - (vol * (Math.Sqrt(exp)))
        (spot * ncdf d1) - (strike * Math.Pow(Math.E, -rate*exp)*(ncdf d2)), ncdf d1
    


Notice that the function returns a tuple of (value,delta). Give that the greeks are available analytically we may as well return them at the same time as the value.

We can poke this a bit to check that it's doing the right thing. A call option that's way in-the-money (i.e. spot > strike) near to expiry, has a lot of intrinsic value:

    
    
    > call 10.0 100.0 0.0 0.0 0.1 0.5;;
    val it : float * float = (90.0, 1.0)
    


An at-the-money call option should have a delta close to 0.5 at expiry:

    
    
    > call 10.0 10.0 0.0 0.0 0.1 0.5;;
    val it : float * float = (0.6301280626, 0.5315064031)
    


All seems fine.

So lets see what we can do about visualising these relationships.


### Using WPF with F#


You'd be forgiven for thinking that the XAML markup language is the only way to construct user interfaces in WPF. Indeed, if you want to avoid writing code it's the way to go. But unfortunately it's not possible to use F# as the code-behind XAML files, so you're forced to use C#. And it's also not as dynamic or immediate as it could be; involving the usual write, compile, run iteration. We can do better in F# using scripts and F# interactive (FSI).

The key is to construct the WPF object model imperatively, rather than declaratively with XAML. Yeah, I know this is normally the opposite of what you'd want to do.

For instance, here's some code to create a Canvas. Note the use of a construction expression to both create the object and set some of it's properties. This is just syntactic sugar which helps to make the object _look_ slightly less mutable than it actually is:

    
    
        let c = new Canvas(Name="Canvas", Width=250.0, Height=250.0)
        c.Background <- SolidColorBrush(Color.FromRgb(228uy,228uy,228uy))
    


We can use the WPF canvas panel to enable us to explicitly place points (or rather, Rectangles) on our plot, it uses absolute locations rather than the implied flow layout of the other containers (Grid, StackPanel etc). Putting this together, we can write a function which will take a parent Panel, and a list of (x,y) tuples and create a plot:

    
    
    let plot (panel:Panel) (pts:(double * double) list) =
        let bgc = new Canvas(Name="Canvas", Width=300.0, Height=300.0)
        let c = new Canvas(Name="Canvas", Width=250.0, Height=250.0)
        c.SetValue(Canvas.LeftProperty, 25.0)
        c.SetValue(Canvas.TopProperty, 25.0)
        c.Background <- SolidColorBrush(Color.FromRgb(228uy,228uy,228uy))
        let x,y = pts |> List.unzip
        let xmin,xmax = x |> List.min, x |> List.max
        let ymin,ymax = y |> List.min, y |> List.max
        let xscale = c.Width / (xmax-xmin)
        let yscale = c.Height / (ymax-ymin)
        pts
        |> List.map (fun pt ->
            let p = new Rectangle(Width=2.0,Height=2.0)
            p.Stroke <- new SolidColorBrush(Color.FromRgb(0uy,0uy,0uy))
            p.SetValue( Canvas.LeftProperty, (fst pt - xmin) * xscale)
            p.SetValue( Canvas.TopProperty, c.Height - (snd pt - ymin) * yscale)
            p
            )
        |> List.iter (fun rect ->
            ignore <| c.Children.Add rect)
        ignore <| bgc.Children.Add (fst (makeYScale ymin ymax))
        ignore <| bgc.Children.Add (fst (makeXScale xmin xmax))
        ignore <| bgc.Children.Add c
        ignore <| panel.Children.Add bgc
        ()
    


The function returns nothing (unit in F# parlance) and performs a side-effecting modify of the passed Panel. I've omitted the makeXScale and makeYScale functions, as they're a bit gnarly, and don't really serve to demonstrate anything in particular.

Now we need to create some data to plot. We can do this in many ways, but one thing I looked at was creating a set of call option values with varying time to expiry. I thought this could use a floating point sequence, but it generated a deprecation warning, so I decided to do it long hand instead. This function returns a list of (time,value) tuples, where time range from `rfrom` to `rto` in `num` steps and value is calculated using the passed function `f`.

    
    
    let varyExpiry f num rfrom rto =
        List.init num (fun n ->
            let s = rfrom + ((float n/float num) * (rto-rfrom))
            float s, fst <| f 10.0 10.0 0.05 0.0 s 0.5 )
    



We can pass the resulting list directly to our plot function, but first we need to create a window to display it in:

    
    
    let w = new Window(Name="Plot",Width=500.0,Height=500.0)
    let wp = new WrapPanel()
    w.Content <- wp
    w.Visibility <- Visibility.Visible
    


Executing this code in FSI will display a blank window (probably behind your active window if you're within Visual Studio). Then we can add the plot to the graph:

    
    
    plot wp (varyExpiry call 100 10.0 100.0)
    


Voila! Obviously the plot is very, very basic, but it's an interesting experiment, and it does makes it extremely easy to plot functions interactively in almost the same way as you can with applications like Excel and MatLab. Pretty powerful.
[caption id="attachment_190" align="alignnone" width="150" caption="Call option values for varying expiries"][![Call option values for varying expiries](build/gatsby/www.ianvoyce.com/assets/2009-06-26-black-scholes-option-pricing-using-fsharp-and-wpf_graph_call_expiry-150x150.png)](http://72.47.193.211/wp-content/uploads/2009/06/graph_call_expiry.png)[/caption] [caption id="attachment_191" align="alignnone" width="300" caption="Call and put option values for varying expiries"][![Call and put option values for varying expiries](http://www.ianvoyce.com/wp-content/uploads/2009/06/graph_call_put_expiry-300x170.png)](http://72.47.193.211/wp-content/uploads/2009/06/graph_call_put_expiry.png)[/caption]

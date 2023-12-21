---
date: 2009-11-24 23:44:55+00:00
description: ''
excerpt: An example of generating random numbers in F# and visualising their distribution
  using the WPF charting control.
featuredImage: build/gatsby/www.ianvoyce.com/assets/2009-11-24-generating-and-plotting-random-numbers_ir
slug: /generating-and-plotting-random-numbers
template: blog-post
title: Generating and plotting random numbers
categories:
- .NET
- F#
- Finance
- Visual Studio
- WPF
tags:
- .NET
- F#
- Finance
- Visual Studio
---

For a while now I've been planning to write a blog post about pricing financial instruments using Monte Carlo techniques in F#. As part of this I needed to generate normally distributed random numbers, and while putting together the code to do it I realised it was interesting enough to warrant its own post.

I'm making use of a few F#/.NET idioms to make the process easier. For instance, sequences (.NET's `IEnumerable`) are used as the source of uniformly distributed random numbers. Also, to verify that the resulting numbers are actually normally distributed, we can easily use existing WPF/silverlight controls to visualise the values direct from within Visual Studio, in a manner similar to [this previous post](http://www.ianvoyce.com/index.php/2009/06/26/black-scholes-option-pricing-using-fsharp-and-wpf/) - but without having to write the plotting code ourselves.
<!-- more -->


### Implementation


So firstly, we create the random number stream:

    
    
    let rr = System.Random(System.DateTime.Now.Minute)
    let rands = seq { while true do yield rr.NextDouble() }
    


As you can see `rands` is an infinite sequence. As such, you should be wary of the "eager" functions from the Seq module that attempt to consume the entire sequence at once, they'll obviously cause a problem here. Also notice that I've picked a somewhat dubious seed for the random number generator, as this is only for demonstration purposes.

In order to obtain normally distributed random numbers I'm using the [Box-Muller](http://en.wikipedia.org/wiki/Box–Muller_transform) transform. I started off with an implementation transcribed from VBA (from [Wilmott](http://www.amazon.com/gp/product/0470319585?ie=UTF8&tag=wwwvoycecom-20&linkCode=as2&camp=1789&creative=9325&creativeASIN=0470319585)![](http://www.assoc-amazon.com/e/ir?t=wwwvoycecom-20&l=as2&o=1&a=0470319585) et al) - yes, yes, I know - but of course this used mutable references. Not nice. So I reworked it a little to use `Seq.pick`, which takes items from the sequence while a predicate returns `None`:

    
    
    let boxMuller _ =
        let (x,dist) = 
            rands
            |> Seq.pick (fun x ->
                let x = 2.0 * x - 1.0
                let y = 2.0 * ((Seq.take 1 rands) |> Seq.hd) - 1.0
                match (x * x + y * y) with
                    | dist when dist < 1.0 -> Some (x,dist)
                    | dist -> None
                ) 
        x * Math.Sqrt(-2.0 * Math.Log(dist) / dist)
    


It's not particularly efficient as we're consuming two uniform numbers to generate one normal one - hence the additional Seq.take within the pick. Of course we could change the function to return the 2 generated numbers as an extra element in the tuple.

Next, let's generate some numbers, and get them in a suitable form to display:

    
    
    let makeData _ =
        Seq.init 100000 boxMuller
        |> Seq.countBy (fun d -> 
            let d = decimal d
            Math.Round(d, 1))
        |> Seq.sortBy fst
    


We initialise a sequence with an arbitrary amount of numbers (100000, enough to get a good distribution, hopefully), then use `Seq.countBy` to do a naive grouping that allows us to see how the numbers are distributed, i.e. counting how many numbers fit in each 0.10 bucket. We then sort the output by bucket. 



### Visualisation


Now we can display what we've got. We can create a simple WPF object-model using imperative code, including the very useful chart control from the WpfToolkit. The chart control enables us to add a series and set our `seq<decimal * float>` directly as the series ItemsSource, which is the standard way to specify the items for a collection control. If you specify the tuple `Item1` and `Item2` properties for the bindings, WPF will be able to access the data from each element, and that's all you need. 


    
    
        let series = 
            ColumnSeries(
                IndependentValueBinding = Data.Binding("Item1"),
                DependentValueBinding = Data.Binding("Item2"),
                ItemsSource = makeData ())
        let chart = Chart()
        chart.Series.Add series
        Window(
            Name="Plot",
            Title="Normally distributed random numbers",
            Width=900.0,
            Height=700.0,
            Content=chart,
            Visibility=Visibility.Visible)
    



Here's the resulting output:
![dist](http://www.ianvoyce.com/wp-content/uploads/2009/11/dist-300x233.png)As I've mentioned before, this is a fantastically immediate way of doing numerical development in Visual Studio. You can enter all of this code directly into F# interactive within VS, iterate, and quickly see the effect that your changes have. It's just as interactive as using the graph features in Excel, and the resulting code is much more easily reusable.

XD5RRV3XDNHN 


    ![](http://www.assoc-amazon.com/s/noscript?tag=wwwvoycecom-20)


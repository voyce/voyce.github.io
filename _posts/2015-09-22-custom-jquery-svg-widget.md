---
date: 2015-09-22 10:58:14+00:00
description: ''
featuredImage: build/gatsby/www.ianvoyce.com/assets/2015-09-22-custom-jquery-svg-widget_svgprogress1.png
slug: /custom-jquery-svg-widget
template: blog-post
title: Custom jQuery SVG widget
categories:
- Graphics
- Javascript
tags:
- javascript
- jQuery
- SVG
- UI
---

I wrote a while back about combining [D3 and Knockout](http://www.ianvoyce.com/index.php/2013/06/23/dynamic-d3-with-knockout-js/). Now in the continuing spirit of web UI mix-and-match, I'm going to try creating something that allows us to leverage SVG within a custom jQuery UI widget.
<!-- more -->


#### Why SVG?


SVG is pretty cool. It started off as a standards-based open vector file format to rival Adobe and Microsoft's versions, but rather than being page layout and printing focused (a la postscript) it adds interactivity and animation features. Various vector illustration packages (including Inkscape and Adobe Illustrator) can export SVG files, and they are notionally human-readable, being XML. But we all know how likely we are to read a machine-generated XML file... 

Browser support has been patchy in previous generations, requiring various plug-ins and hoop jumping, but now it's [supported](http://caniuse.com/#feat=svg) in all the widely used versions, including on mobile.

One aspect that makes it especially powerful is its "retained mode" object model and the integration this enables with other web standards such as CSS. 

Retained mode means that the hierarchy of objects making up the SVG element are still accessible after it's drawn. That means we can find, access and alter aspects of it easily, for animation or styling purposes. SVG is described with the same declarative element/attribute style we're used to from HTML/XML. 

Contrast this with the HTML Canvas tag, where we describe what's to be drawn imperatively in Javascript, and this information is discarded after each frame is rendered; it's generally more efficient in terms of memory usage because of this, but requires a callback or render loop to draw each frame.

If you want to see some excellent examples of the power of SVG, check out [D3](http://d3js.org). 



##### SVG Animation - fly in the ointment


**Danger: Fast-moving Web 'standards' ahead!** So, I thought it would make sense to use the native SVG animation support, known as **SMIL** (Synchronised Multimedia blah whatever), but had real problems adding `<animate>` elements via jQuery, and then realised that it's already deprecated in the current version of Chrome.

The deprecation message recommends using **Web Animation** instead. But it turns out that's not widely supported at all, as reported by [caniuse.com](http://caniuse.com/#feat=web-animation), so not really a viable option.

I decided to take a look at what the other frameworks implement their animation support. It turns out that D3 and [Snap.svg](http://snapsvg.io) use their own mechanisms, including providing a suite of easing functions and all the required machinery for timing and redrawing. There's also a helpful API exposed by browsers to enable this kind of arbitrary animation: `requestAnimationFrame`. 

This gives us an efficient hook into the browser repaint mechanism, where we get called back to make our changes just prior to the screen being redrawn. More details [here](http://creativejs.com/resources/requestanimationframe/). Now we can side-step the compatibility issues, avoid introducing a new dependency and roll our own simple animation. 



#### Why a custom control?


jQuery.UI provides a whole host of standard controls out-of-the-box. But we all know there's always _some_ special, unique functionality that your app needs. For small modifications and refinements you can e.g. style your widgets using CSS, handle events in certain ways to alter its behaviour, but for more in-depth changes, or entirely new behaviours, it's also possible to create a whole new custom control.  



#### Enough "why", let's do it!


jQuery custom widgets are created by the widget factory in jQuery.UI. The widget itself consists of an identifier (`namespace`.`name`) and a set of functions that define the behaviour.
 
The most basic widget definition would consist of simply:

    
    
    $.widget( "voyce.svgprogress", {});
    


And a DOM node in the HTML on which to hang the object:

    
    
    <div id="#main"/>
    


We can then associate the Javascript function with the node in the standard jQuery way: (note that we're using the second part of the identifier we specified in the `widget` function):

    
    
    $("#main").svgprogress();
    


The widget base provides support for various pieces of boilerplate which you'd otherwise have to write yourself: including standard construction and dealing with specifying options.

As far as a bare-bones implementation goes, that's pretty much it, it's enough to have jQuery associate the function with the DOM node, but there's nothing to see yet. Let's add some actual functionality.

I'm going to implement a very simple circular progress bar. It's a bit of a contrived example, and there are probably a variety of ways you could implement it in pure CSS, but I'm determined to try the SVG version.

[![jQuery SVG progress widget](build/gatsby/www.ianvoyce.com/assets/2015-09-22-custom-jquery-svg-widget_svgprogress1.png)](build/gatsby/www.ianvoyce.com/assets/2015-09-22-custom-jquery-svg-widget_svgprogress1.png)

Let's add a function to create the elements that will actually display the data. Unsurprisingly the `_create` function is what's used by the widget factory when your control is instantiated. You can see how we're building up the DOM with our simple hierarchy of SVG elements:

    
    
    _create: function() {
        // Set the standard classes.
        this.element
             .addClass("ui-widget ui-widget-content");
    
        // The SVG parent container.
        this.container = $(SVG("svg"))
            .appendTo(this.element)
            .attr("width", "100%")
            .attr("height", this.options.height);
        
        // The arc path.
        this.path = $(SVG("path"))
            .appendTo(this.container)
            .attr("class","svgprogress")
            .attr("height", "100%");
    
        // Display a text version of the progress
        // as a percentage.
        this.txt = $(SVG("text"))
             .appendTo(this.container)
             .attr("class","progresstext")
             .attr("x", this.options.height/2)
             .attr("y", this.options.height/2)
             .attr("text-anchor", "middle")
             .attr("alignment-baseline", "middle");
    
        this.oldValue = 0;
        this._refreshValue(this.oldValue);
    },
    


Obviously in the interest of testability/model-view separation, we'd want to specify as few visual aspects in here as possible. They should instead be controlled by the CSS in the presentation layer, but I'm afraid we are making a few assumptions (enforce aspect ratio, place text centrally in control).

One other thing you might notice is the use of a `SVG` function to create the element. The purpose of this is to assign it the correct namespace. As far as I'm aware it's not possible to do this directly with jQuery, and unfortunately SVG lives in a namespace separate to the rest of the DOM. That, combined with the case insensitivity of jQuery attribute names, can make it a little awkward to control SVG using jQuery functionality. It's all a bit unfortunate.



#### Animating the SVG path


The meat of the control is the function that calculate the point on the arc, and moves between the old and new values. For simplicity I'm doing this linearly, you can of course choose to implement whatever funky easing function you fancy.

    
    
    _refreshValue : function() {
        var height = this.options.height;
    
        // Create an SVG path data string
        var generatePath = function(max, from, to, progress){
            var centre = height / 2;
            var radius = height * 0.8 / 2; // Leave a bit of surrounding space
            var startY = centre - radius;
    
            var value = from + ((to - from) * progress);
    
            var deg = Math.min(((value/max) * 360), 359.9);
            // Subtract 90, because we want to start from the top
            // not the RHS
            var radians = Math.PI*(deg - 90)/180;
            var endx = centre + radius * Math.cos(radians);
            var endy = centre + radius * Math.sin(radians);
            var isLargeArc = deg > 180 ? 1 : 0;  
    
            return "M"+centre+","+startY+" A"+radius+","+radius+" 0 "+isLargeArc+",1 "+endx+","+endy;
        };
    
        var initial_ts = new Date().getTime();
        var duration = 125; // Run for 1/8th second
        var handle = 0;
        // Capture instance variable values
        var vfrom = this.oldValue;
        var vto = this.options.value;
        var max = this.options.max;
        var path = this.path;
    
        // Callback for each animation frame
        var draw = function() {
            var progress = (Date.now() - initial_ts)/duration;
            if (isNaN(vfrom))
                vfrom = vto;
            if (progress >= 1) {
                window.cancelAnimationFrame(handle);
            } else {
                var newPath = generatePath(max, vfrom, vto, progress);
                path.attr("d", newPath);
                handle = window.requestAnimationFrame(draw);
            }
        };
        draw();
    
        // Set textual version of progress too
        this.txt.text(Math.round((vto/max)*100) + '%');
    }
    

  
This is a pretty standard bit of geometry to calculate the point on a circle that corresponds to a certain angle. The SVG path is defined using these parameters:
<table >
<tr >
<td >M
</td>
<td >Move to start point
</td>
<td >The top of the circle
</td></tr>
<tr >
<td >A
</td>
<td >Arc path: radius of circle
</td>
<td >The same for each radius, as we want a line, not a fill
</td></tr>
<tr >
<td >
</td>
<td >x-axis rotation
</td>
<td >Always 0 in our case
</td></tr>
<tr >
<td >
</td>
<td >large-arc
</td>
<td >Whether we want to draw the large or small part of the circle
</td></tr>
<tr >
<td >
</td>
<td >sweep
</td>
<td >Always 1 for our case
</td></tr>
<tr >
<td >
</td>
<td >end x,end y
</td>
<td >Calculated end position on the circle
</td></tr>
</table>

Wow, that was fun. Now we have a widget we can instantiate:

    
    
    $("#main").svgprogress({
    	max : 200,
    	value: 10,
    })
    


And we can use jQuery's standard `bind` mechanism to listen to events:

    
    
    .bind("click", function(){
    	// Move it on by 10 units whenever we click
    	$(this).svgprogress("value", $(this).svgprogress("value") + 10);
    })
    


Although the clunky format for getting/setting values is a bit unfortunate, it's apparently by design in the widget framework to "prevent pollution of the jQuery namespace while maintaining the ability to chain method calls". You can make it marginally less fugly by getting the associated jQuery factory object from the DOM element and then calling via it:

    
    
    var o = $("#main").data("voyce-svgprogress");
    o.value(20); 
    



If you want to see if in action, check out the [demo here](/demos/jQueryWidget/index.html). Or grab the code from [github](https://github.com/voyce/jQueryWidget). Enjoy!

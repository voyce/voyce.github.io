---
date: 2013-06-22 23:45:06+00:00
description: ''
featuredImage: ''
slug: /dynamic-d3-with-knockout-js
template: blog-post
title: Dynamic D3 with Knockout.js
categories:
- Graphics
- Software Development
tags:
- d3
- javascript
- knockout
- wpf
---

A couple of things happened recently that prompted me to write this blog post. Firstly, I've been playing around with HTML5/javascript based user interfaces and data visualisation. Secondly, I watched a [fascinating presentation](http://vimeo.com/66085662) from UX guru [Brett Victor](http://worrydream.com), making me wonder if it was possible to create an interactive, data-drawing app like the one he demonstrates, purely with Javascript. There are 2 well established JS frameworks that we could combine to help us here: [Knockout.js](http://knockoutjs.com) and [D3](http://d3js.org). But can we make them work well together? 
<!-- more -->


### The pieces - Knockout


I assume you know roughly what Knockout.js and D3 are. Briefly, Knockout provides a declarative way of building data-bound user interfaces. It allows you to create "observable" properties on your model that raise change events and cause your bound UI elements to be automatically updated. For anyone familiar with Microsoft's UI frameworks WPF and Silverlight, this is equivalent to its `INotifyPropertyChanged` and binding mechanism.

Here's a trivial view model that exposes an observable string:

    
    
    var viewModel = { 
        foo : ko.observable("doodah")
    };
    ko.applyBindings(viewModel);
    


Which we can then bind to something in our view/HTML:  

    
    
    Ooh look, foo is <span data-bind="text: foo"></span>
    


Now whenever the view-model is updated, either in code (`viewModel.foo("blah")`) or by binding it to a control that accepts input, our UI will automatically reflect the change.

Other important features in KO are [computed observables](http://knockoutjs.com/documentation/computedObservables.html) - observable properties calculated based on other observable or vanilla properties - and templating - the ability to generate elements based on data. But we won't be making extensive use of either of these things here.

Of course, as your view model is just a normal JS object, you can call its functions to change its state, just as you'd do with `ICommand` in WPF. We're going to use this to hook up a button in the UI that adds an element to an array.

As a slight aside, I must say that having worked with a large, immensely clever but complicated framework for WPF written in F#/.NET, Knockout feels very lightweight yet still powerful. It includes much of the same type of functionality, but in just a few KB of Javascript!


### D3


D3 is a library for creating data-driven documents. In practice, it provides a declarative way of mapping Javascript objects to visual HTML objects in the DOM. Declarative here means that we provide the rules on how to generate and remove elements as required, rather than doing it explicitly. For instance:

    
    
    d3.select("body").selectAll("p")
        .data([4, 8, 15, 16, 23, 42])
      .enter().append("p")
        .text(function(d) { return "I’m number " + d + "!"; });
    


This somewhat cryptic string of instructions tells D3 to `select` all of the `p` elements in the DOM, and join them with the specified `data` (here, a static array). Where there are more data points than elements, D3 will add an element. These new elements `enter` the DOM by appending a `p`, and setting the text of that to the result of the specified function, evaluated with the corresponding data element (here, the array element).

Phew.

Of course, that's just scratching the surface. D3 also has extensive facilities for describing transitions, modifying the DOM, scaling and managing axes and interacting with elements via behaviours.



### Putting it together


So, how can we combine these two immensely powerful libraries to do something interesting? Or at least vaguely interactive.

Let's see if we can use D3 to draw some rectangles, described in a Knockout view-model, which we can modify using D3 or by altering the data itself. 

There is some overlap in functionality here: KO enables us to generate DOM content based on data, as does D3. In theory we could use a KO template to generate fragments of SVG markup for each of our view models. But that would involve generating the elements long-hand and we wouldn't have the association between view and view-model. Here's what we're _not_ going to do:

    
    <code>
    <!-- Generate an SVG rect for each data item in our view model -->
    <svg id="svg" width="500px" height="500px">
        <g data-bind="foreach: rects" id="rects">
            <rect data-bind="attr:{width: w, height:h, x:x, y:y}" opacity="0.3"/>
        </g>
    </svg>
    </code>


If we used this approach we wouldn't have an easy way to update the view model when the view is updated - the reverse of what we'd normally do - say, by dragging a DOM element directly.

To start let's define our view model as an array of rectangles: 

    
    
    function Rect() {
       var self = this;
       self.x = ko.observable(0);
       self.y = ko.observable(0);
       self.w = ko.observable(100);
       self.h = ko.observable(100);
       self.name = ko.observable(makeName());
    };
    function ViewModel() {
        var self = this;
        self.rects = ko.observableArray([]);
        self.addRect = function () {
            self.rects.push(new Rect(self));
        };
    };
    ko.applyBindings(new ViewModel());
    


We've even got an `addRect` function that will push a new instance into our array. It's worth noting that we're using `push` on the observableArray, not on the underlying array, i.e. we're _not_ dereferencing rects by doing `self.rects**()**.push...`. This is important because doing so will mean that no knockout notifications are raised (believe me, I spent a while trying to figure that out!). 

Now, we can pass the same ViewModel to D3, providing 4 sets of "rules": 

    
    
    // 1) Join the existing SVG rectangles with our data:
    var rects = d3.select("#svg")
        .selectAll("rect")
        .data(d, function (d) { return d.name(); });
    // 2) For new data, add an SVG rect element and set its id
    rects.enter()
        .append("rect")
        .attr("id", function (d) { return d.name();});
    // 3) For existing data, update the elements x, y, width and height  
    rects.attr("x", function (d) { return d.x(); })
        .attr("y", function (d) { return d.y(); })
        .attr("width", function (d) { return d.w(); })
        .attr("height", function (d) { return d.h(); })
        .call(drag);
    // 4) For removed data, remove the element
    rects.exit().remove();
    


This is pretty nice. It makes use of D3's optimisation of element creation; it (and us) want to avoid adding elements to the DOM unnecessarily, so we provide a data keying function, and when D3 finds a matching data element, it updates rather than recreates the corresponding visual.

Now, we need a way of getting the KO data fed to D3 at the right time: when it's updated.

One of the first things to notice is that if we observe the array of rects, we only see changes to the array, not to its elements. In other words, we only know when items are added or removed, not when elements properties are changed. Seeing as we need to know when a rectangle's position changes, we'll need to do some more work.

There are a few solutions around for providing "dirty flags" for KO view models. I decided to use [this one](http://www.knockmeout.net/2011/05/creating-smart-dirty-flag-in-knockoutjs.html). It gives us the ability to find out when any of the observables change. We can add a dirty flag property to our view model like this:

    
    
        self.dirtyFlag = new ko.dirtyFlag(self);
    


Then subscribe to it like this (where item is an instance of our view model):

    
    
        item.dirtyFlag.isDirty.subscribe(function () {
            // Do something!
        }
    



In actual fact this is overkill for our case. It would be more suitable if our view models had many properties that we didn't want to track individually. Instead we can create a single property computed from the ones we're interested in and subscribe just to that.  

    
    
        self.rect = ko.computed(function(){
            // In our case it doesn't matter we return; this function just needs to be
            // something that reads the values of the properties we're interested in
            return {x:self.x(), y:self.y(), w:self.w(), h:self.h()};
        });
    


So now we can subscribe to the updates we care about using a subscribe on the array, and on each item in it. Note a couple of things: 1) the function we pass to subscribe is always invoked with the entire array, rather than just the added or removed items (as happens with WPF's [INotifyCollectionChanged](http://msdn.microsoft.com/en-us/library/system.collections.specialized.inotifycollectionchanged.aspx)) and 2) we keep track of the subscriptions we add, so that they can be subsequently removed. 



### The result


So, we can write a function that is passed our view model data, and then gets called whenever the view model is changed, either programatically or via user interaction:

    
    
                function update(data) {
                    // Join elements with data
                    var rects = d3.select("#svg")
                        .selectAll("rect")
                        .data(data, function (d) { return d.name(); });
                    // Create new elements by transitioning them in
                    rects.enter()
                        .append("rect")
                        .attr("id", function (d) { return d.name(); })
                        .attr("opacity", 0.0)
                        .transition()
                        .duration(1000)
                        .attr("opacity", 0.5);
                    // Update existing ones by setting their x, y, etc
                    rects.attr("x", function (d) { return d.x(); })
                        .attr("y", function (d) { return d.y(); })
                        .attr("width", function (d) { return d.w(); })
                        .attr("height", function (d) { return d.h(); })
                        .call(drag);
                    rects.exit().remove();
                }
    
                var subs = []; // for keeping track of subscriptions
                // Listen for changes to the view model data...
                vm.rects.subscribe(function (newValue) {
                    update(newValue);
                    // Dispose any existing subscriptions 
                    ko.utils.arrayForEach(subs, function (sub) { sub.dispose(); });
                    // And create new ones...
                    ko.utils.arrayForEach(newValue, function (item) {
                        subs.push(item.rect.subscribe(function () {
                            update(newValue);
                        }));
                    });
                });
    


Let's generate some HTML that will let us create new rectangles, and see the changes (as we drag the D3 rects) and set the values (by typing into the controls):

    
    
            <button data-bind="click:addRect">Add</button>
            <div data-bind="foreach: rects">
                x:<input data-bind="value: x" size="6"></input>
                y:<input data-bind="value: y" size="6"></input>
                w:<input data-bind="value: w" size="6"></input>
                h:<input data-bind="value: h" size="6"></input>
                <br></br>
            </div>
    


You can see it in action [here](http://www.ianvoyce.com/examples/d3ko/draw.html).

There we have it, a fairly simple way of tying-up D3-generated visuals with Knockout-driven data. It's obvious that we're just scratching the surface here, but when your view model gets more complicated, KO will really come into it's own, cascading updates and managing interaction between different parts of the data model.    

Check out a Gist of the source [here](https://gist.github.com/voyce/5842759).

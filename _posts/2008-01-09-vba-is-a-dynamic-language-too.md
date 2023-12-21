---
date: 2008-01-09 22:31:52+00:00
description: ''
featuredImage: ''
slug: /vba-is-a-dynamic-language-too
template: blog-post
title: VBA is a dynamic language too!
categories:
- Software Development
---

There’s lots of talk about dynamic languages recently; what with the ubiquity of Javascript, and the rise and rise of Python, Ruby etc, and now Microsoft jumping into the fray with the DLR - dynamic language runtime - to make creating a dynamic language almost as easy as using one.But sometimes, we forget the old boys, sitting quietly in the corner, like poor, aged old VBA. Once the poster boy for scripting, and still prevalent in even the most recent versions of the Microsoft Office apps, but now really just biding it’s time, waiting for it’s inevitable replacement by some .NET-based upstart.One of things people often talk about with Javascript is that you can dynamically add properties to an object at runtime, e.g.:



var circ = new Object();
circ.hittest = function(x,y) { return false; };

But, people often forget that you can do a similar thing with VBA in Excel. Yes, VBA.
For instance, add a function to a worksheet object:



	
  * Open the VBA editor (Alt+F11)

	
  * Double click on Sheet1

	
  * Add some code:




Sub NewMethod()
MsgBox “I got called!”
End Sub

Now it’s possible to call this method as if it existed directly on the Worksheet class! For example, with some code in the Workbook like this:



Sub CallIt()
Worksheets(”Sheet1″).NewMethod
End Sub

So there we go, VBA was dynamic (thanks to IDispatch) before it was even cool…

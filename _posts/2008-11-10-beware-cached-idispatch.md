---
date: 2008-11-10 11:32:15+00:00
description: ''
featuredImage: ''
slug: /beware-cached-idispatch
template: blog-post
title: Beware cached IDispatch
categories:
- .NET
- COM
- Debugging
- Excel
- F#
tags:
- Excel
- F#
- IDispatch
- VBA
- VBScript
---

I've kinda given it away there with the title, but we had an interesting set of symptoms exhibited the other day while trying to call a function in an Excel workbook via F#. It appeared that the function being called would fail depending on what had been called previously. Very odd.

A bit of background: as you may know, if you add functions to the worksheet or workbook code in Excel then they appear as callable methods on the objects themselves. This is achieved with the use of dynamic dispatch and IDispatch. For example, creating a workbook with this function in it's VBA code:

`Public Function Foo() As Double
    Foo = 100
End Function`

Means you can call it like this:

`MsgBox CStr(ThisWorkbook.Foo)`

As well as being able to call it like this from within the Excel session (i.e. in other VBA code in the process), you can also access it externally using the COM object model that Office applications expose. For instance, you can use VBScript:

`Dim excel, wkb
Set excel = CreateObject("Excel.Application")
Set wkb = excel.Workbooks.Open("a.xls")
WScript.Echo wkb.Foo
`

Or, more interesting, using F#:





    let excel = new Excel.ApplicationClass()




    let wkb = excel.Workbooks.Open(@"c:\a.xls")




    wkb.GetType().InvokeMember("Foo", BindingFlags.InvokeMethod, null, wkb, null)






The key part here is that we're using wks.GetType() to get a managed representation of the unmanaged Excel COM interface. Under the covers this is creating a runtime callable wrapper (RCW) to wrap the worksheet COM object.

However, the problem we were seeing was that opening multiple sheets resulted in failures to call the method in certain situations. Although the VBA signature was exactly the same in all of the sheets, it seemed that opening b.xls after a.xls, would fail; returning null when we expected it to return a value. If we opened c.xls after b.xls, it would fail in a different way; never actually making it to the body of the function. Very odd.

My first suspicion was that it was somehow related to COM object vs .NET object lifetime. This is quite a common problem whens invoking Excel using managed code. It's bad mixing COM and .NET anyway; generally deterministic, reference-counted lifetime semantics don't play well with the garbage collector. Throwing an app with a full-blown UI being managed as COM object into the mix just complicates matters further. Anyway, it's been widely discussed, so I won't say any more about it here; suffice to say that calling Marshal.ReleaseCOMObject and GC.Collect got us to the point where we could see the Excel process terminating, so we knew that the failure wasn't due to some state being cached inside there.

So we concentrated on different aspects of the problem:



	
  * Given the pattern of failures, it seemed that the order of opening the sheets and calling affected the outcome. This hinted that something was persisting betweeen calls, but not in the client (Excel) site.

	
  * The code had previously worked when written in VBScript, so there was nothing intrinsically wrong with the operations we were performing.


This seemed to strongly indicate that something was being cached at the .NET level. And the major difference between the .NET code and the VBScript was that the former used Type.GetType() on the worksheet object to get it's managed representation, while the latter used the IDispatch directly.

So it looked like GetType() was caching some information about the particular IDispatch implementation that it encountered first, then reusing that for subsequent worksheet implementations which actually had a slightly different layout, i.e. although they also had the Foo function which we were trying to call, they had a different set of other dynamic functions too.

After a bit of digging about I uncovered this gem: [the mapping between interface pointers and runtime callable wrappers](http://blogs.msdn.com/mbend/archive/2007/04/18/the-mapping-between-interface-pointers-and-runtime-callable-wrappers-rcws.aspx). Which seemed to describe exactly what we were seeing. The first time through the loop, the runtime is asked to get the type and is given a COM pointer, for which it creates a RCW that we can use to invoke Foo. The second time through the loop, the runtime thinks it's seen the object before, so rather than perform the expensive operation of creating the RCW again, it just returns the original one. The problem is, the underlying COM object is different!

So, in order to prevent the runtime for trying to cache the RCW, we need to use [Marshal.GetUniqueObjectForIUnknown](http://msdn.microsoft.com/en-us/library/system.runtime.interopservices.marshal.getuniqueobjectforiunknown(VS.80).aspx) and that does the trick nicely. We first need to get an IUnknown for our object, than convert that back into an object, which is actually a RCW:





    let wkb = Marshal.GetUniqueObjectForIUnknown(Marshal.GetIUnknownForObject(wkb))






Although it's less efficent, at least the code works, and it finally allows us to call the dynamic methods on the workbook object from F# .NET.

It will be interesting to see how `dynamic` in .NET 4.0 addresses this kind of issue.

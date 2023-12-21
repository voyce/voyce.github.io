---
date: 2007-05-11 22:29:35+00:00
description: ''
featuredImage: ''
slug: /exposing-static-libraries-to-net-using-ccli
template: blog-post
title: Exposing static libraries to .NET using C++/CLI
categories:
- .NET
- Software Development
---

 




 




I’ve been looking recently at how to make unmanaged C++ code in static libraries available to code written in .NET languages.




There isn’t any direct way of calling into a C++ static library (.lib) from C# code, but this isn’t suprising as they’re very different in terms of how they’re compiled and linked to form an executable. This is where managed C++ - or to give it’s official name C++/CLI - comes in; it does know how to link managed and unmanaged modules within the same binary. So, if you’ve got a static library that you want to make available to .NET clients, there are a couple of options:




 






	
  1. Build the .lib into a standalone unmanaged DLL, and call into it using P/Invoke

	
  2. Build the .lib into a DLL using managed C++


The problem with the first approach is that, especially if you have a complicated method signature, you’re likely to have some complicated marshalling instructions wherever you declare it using [DllImport]. Also, if you plan on updating your DLL at some point, you’ll quickly find yourself back in the pre-COM world of DLL hell - version management becomes a headache.

The second approach gives you some further options about how you expose the functionality to callers:

	
  1. Unmanaged entry points via the normal C++ __declspec(dllexport) and/or .DEF file mechanisms

	
  2. Managed classes that are publicly visible and consumable by .NET clients, using public ref class Foo.

	
  3. A combination of the two!


Exposing managed classes is very straightforward, assuming you’ve already thrown the /clr switch on the project (the binary/wrapper project, not the static library), you can write code like:

    
    public ref class Foo



    
    {



    
    public:



    
        static double Bar(array<double,2> ^ConstraintsLHS, // matrix



    
            array<double> ^ConstraintsLHS) // vector



    
        {



    
            // TODO: call into unmanaged static library function.



    
            // “IJW” interop will deal with marshalling.



    
            return -1;



    
        }



    
    };


This has the advantage that it’s much more natural for .NET clients to consume your code. They can pass managed types to the function, and the “It Just Works” interop in the C++ code will deal with the conversion at the point that you call into the unmanaged function in the static library. It’s also possible to do some more specific marshaling of your own; for instance pinning input arrays to avoid additional copies.

Also users can add references directly to your managed DLL, because it’s an assembly just like any other, and take advantage of all the usual assembly versioning controls.

If you also want to allow unmanaged clients to call your DLL in a natural way, you can use #pragma managed to temporarily switch modes when you define your function:

    
    #pragma managed(push, off)






    
    __declspec(dllexport) double Foo_Bar()



    
    {



    
        // TODO: call into static library function.



    
        // No marshalling/transition required if called by unmanaged code.



    
        return -1;



    
    }






    
    #pragma managed(pop)


Now your function will be callable without incurring a managed/unmanaged transition, although it’s worth noting that your DLL will still have a dependency on MSCOREE.DLL, so it will need to be present on machines where it’s used.

So as you can see, C++/CLI gives you a variety of ways of exposing legacy C++ code in an easily callable way to both managed and unmanaged clients. This gives the developer a high degree of control over how and when marshalling and transitions occur.

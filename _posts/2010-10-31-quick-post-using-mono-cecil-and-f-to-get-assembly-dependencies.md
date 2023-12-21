---
date: 2010-10-31 13:31:43+00:00
description: ''
excerpt: Using Mono.Cecil and F# to list assembly dependencies.
featuredImage: build/gatsby/www.ianvoyce.com/assets/2010-10-31-quick-post-using-mono-cecil-and-f-to-get-assembly-dependencies_Mono-gorilla-aqua.100px.png
slug: /quick-post-using-mono-cecil-and-f-to-get-assembly-dependencies
template: blog-post
title: 'Quick post: Using Mono.Cecil and F# to get assembly dependencies'
categories:
- .NET
- F#
tags:
- .NET
- cecil
- DGML
- dll
- F#
- gac
- mono
---

One of the tools I use a lot when doing C++ development and debugging is ["dependency walker"](http://www.dependencywalker.com/); an app that displays all the static dependencies of an executable. These are dependencies created by referencing functions from an import library (.lib file) at compile time. If any of the imported DLLs are missing at run-time, the executable will fail to load, normally with error 2: file not found. Obviously pretty disastrous in production. The .NET equivalent is the binding failure. You can track down what went wrong at runtime using fuslogvw, but I've often wished for a tool like 'depends' to work out up-front what dependencies are required. Luckily because assemblies includes a list of dependent libraries in the form of a manifest this information can be accessed using reflection.

![Mono-gorilla-aqua.100px](build/gatsby/www.ianvoyce.com/assets/2010-10-31-quick-post-using-mono-cecil-and-f-to-get-assembly-dependencies_Mono-gorilla-aqua.100px.png)I'm a big fan of the [Mono.Cecil](http://www.mono-project.com/Cecil) library for doing reflection (and more!) with .NET. I've had issues in the past where the built-in .NET reflection (using Assembly.ReflectionOnlyLoad) attempts to load dependent libraries as you iterate over exposed types, even though it's not supposed to (unfortunately I don't have a repro to hand). This makes it very difficult to work on an assembly without having all of its dependencies available. Cecil doesn't have this problem because it accesses the assembly in a lower-level way.
<!-- more -->
I downloaded and installed [Mono](http://www.go-mono.com/mono-downloads/download.html) (the latest version is 2.8) and referenced it from the installed Mono GAC location. Then it was just a matter of a handful of lines of F# (after a bit of spelunking through the Cecil API to find the methods relating to assembly references).

    
    
    #r @"C:\Program Files (x86)\Mono-2.8\lib\mono\gac\Mono.Cecil\0.6.9.0__0738eb9f132ed756\Mono.Cecil.dll"
    #r @"gachelper.dll"
    
    open System
    open System.IO
    open Mono.Cecil
    
    let getReferencedAssemblies (asm : string) : string list =
        let ad = AssemblyFactory.GetAssembly asm
        let refs = ad.MainModule.AssemblyReferences
        refs 
        |> Seq.cast 
        |> Seq.fold (fun found (r : AssemblyNameReference) -> 
            let fullPath = ref ""
            match gachelper.GAC.TryGetFullPath (r.Name, fullPath) with
            | false -> found @ [ Path.Combine [|(System.IO.Path.GetDirectoryName asm); r.Name + ".dll"|] ]
            | true -> found @ [!fullPath]        
            ) []
    


As you can see I've also used a little C++/CLI wrapper around the GAC to get access to full paths of installed assemblies (which only works with the Microsoft CLR), but I'll talk about that in a separate post, or you can grab the code [here](http://github.com/voyce/gachelper).

Now, we can call our function to get the set of dependencies, e.g. by using F# interactive, we also happen to be getting the dependencies *of* FSI:

    
    
    > getReferencedAssemblies @"C:\Program Files (x86)\FSharp-2.0.0.0\bin\fsi.exe";;
    val it : string list =
      ["C:\Windows\Microsoft.Net\assembly\GAC_32\mscorlib\v4.0_4.0.0.0__b77a5c561934e089\mscorlib.dll";
       "C:\Windows\Microsoft.Net\assembly\GAC_MSIL\FSharp.Core\v4.0_4.0.0.0__b03f5f7f11d50a3a\FSharp.Core.dll";
       "C:\Windows\Microsoft.Net\assembly\GAC_MSIL\FSharp.Compiler\v4.0_4.0.0.0__b03f5f7f11d50a3a\FSharp.Compiler.dll";
       "C:\Windows\Microsoft.Net\assembly\GAC_MSIL\System.Windows.Forms\v4.0_4.0.0.0__b77a5c561934e089\System.Windows.Forms.dll";
       "C:\Windows\Microsoft.Net\assembly\GAC_MSIL\FSharp.Compiler.Interactive.Settings\v4.0_4.0.0.0__b03f5f7f11d50a3a\FSharp.Compiler.Interactive.Settings.dll";
       "C:\Windows\Microsoft.Net\assembly\GAC_MSIL\FSharp.Compiler.Server.Shared\v4.0_4.0.0.0__b03f5f7f11d50a3a\FSharp.Compiler.Server.Shared.dll";
       "C:\Windows\Microsoft.Net\assembly\GAC_MSIL\System\v4.0_4.0.0.0__b77a5c561934e089\System.dll"]
    



This is, of course, the tip of the iceberg in terms of Cecil functionality; there are lots of far more interesting things you can do - like IL extraction and re-writing - things which are impossible to do with the Microsoft reflection API.

I ended up using this code to generate DGML graphs that can be opened and explored using VS2010. This functionality comes "in the box" with VS2010 Architecture Explorer in the Ultimate Edition - but who can afford that...? We can do much the same ourselves by just using the information we get from Cecil and spitting out DGML directly. The files can be opened read-only in the Premium edition for perusal.

For the curious, here's the (somewhat fugly and imperative) F# code to generate the DGML file. It creates a minimal file that only contains `Link` elements; Visual Studio will "fill in the blanks" by adding the nodes themselves when you open the file.

    
    
    let genDgml asm (out : string) =
        let doc = XmlDocument()
        let nsURI = "http://schemas.microsoft.com/vs/2009/dgml"
        let nsmgr = XmlNamespaceManager(doc.NameTable)
        nsmgr.AddNamespace("", nsURI)
        let root = doc.CreateElement("DirectedGraph", nsURI)
        let links = doc.CreateElement("Links", nsURI)
        let rec genRefs added asm =
            getReferencedAssemblies asm
            |> List.fold (fun added dep -> 
                let link = doc.CreateElement("Link", nsURI)
                link.SetAttribute("Source", Path.GetFileNameWithoutExtension asm)
                link.SetAttribute("Target", Path.GetFileNameWithoutExtension dep)
                ignore <| links.AppendChild link
                if not <| (added |> Map.containsKey dep)
                then genRefs (added.Add (dep, false)) dep
                else added
            ) added
        ignore <| genRefs Map.empty asm
        ignore <| root.AppendChild links
        ignore <| doc.AppendChild root
        doc.Save out
    


[caption id="attachment_1101" align="alignright" width="300" caption="The DGML in cluster view"][![The DGML in cluster view](build/gatsby/www.ianvoyce.com/assets/2010-10-31-quick-post-using-mono-cecil-and-f-to-get-assembly-dependencies_dgml-300x158.png)](http://www.ianvoyce.com/wp-content/uploads/2010/10/dgml.png)[/caption]If you run it against the FSI.exe as we did before it will chug away for a while and then generate this file, which contains all of the dependencies.

It can be quite enlightening, as well as useful, to see your app dependencies laid-bare before you...

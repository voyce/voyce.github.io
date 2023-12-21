---
date: 2010-03-31 17:31:15+00:00
description: ''
excerpt: It's easy to call MSBuild tasks directly from F#. Although possibly unnecessary.
featuredImage: build/gatsby/www.ianvoyce.com/assets/2010-03-31-calling-msbuild-tasks-with-fsharp_msbuild_verbosity-300x174.png
slug: /calling-msbuild-tasks-with-fsharp
template: blog-post
title: Calling MSBuild tasks with F#
categories:
- .NET
- F#
- Software Development
- Visual Studio
tags:
- F#
- msbuild
- Visual Studio
---

The other day I was trying to understand some strange behaviour in msbuild with regard to how it resolves referenced assemblies. I thought I'd try directly invoking the tasks that are used during the build, specifically [ResolveAssemblyReference](http://msdn.microsoft.com/en-us/library/9ad3f294.aspx), so that I could experiment with them in F# interactive. It turned out to be pretty straightforward.
<!-- more --> 
In this example I'm attempting to resolve a reference to "class1.dll" from the GAC, taking into account the contents of the app.config (which can be used change the version resolution in MSBuild):

    
    
    #I @"C:\winnt\Microsoft.NET\Framework\v2.0.50727\"
    #r @"Microsoft.Build.Tasks.dll"
    #r @"Microsoft.Build.Utilities.dll"
    #r @"Microsoft.Build.Framework.dll"
    #r @"Microsoft.Build.Engine.dll"
    
    open System
    open Microsoft.Build.Framework
    open Microsoft.Build.Utilities
    open Microsoft.Build.Tasks
    
    do
        let t = 
            ResolveAssemblyReference(Silent = true,
                Assemblies = [| upcast TaskItem( ItemSpec="class1" ) |],
                AllowedAssemblyExtensions = [|".dll"|],
                AppConfigFile = "c:\\dev\\myapp\\myapp.exe.config",
                SearchPaths = [|"{GAC}"|]
                )
        match t.Execute() with
        | true  -> printf "%A\n" t.ResolvedFiles
        | false -> printf "Task failed"
    


The output is:

    
    
    [|C:\WINNT\assembly\GAC_MSIL\class1\1.1.8.0__120bce90f68b861a\class1.dll|]
    


The code makes good use of F#'s named constructor arguments, which map to property setters on the object we've just created. We can also create the arrays of [TaskItem](http://msdn.microsoft.com/en-us/library/microsoft.build.utilities.taskitem.aspx)s and strings fairly easily, although an explicit upcast is required in order to obtain the ITaskItem (F# doesn't upcast implicitly like many object oriented languages).

I've just discovered that Jomo Fisher wrote about [exactly the same thing](http://blogs.msdn.com/jomo_fisher/archive/2008/05/22/programmatically-resolve-assembly-name-to-full-path-the-same-way-msbuild-does.aspx) back in May 2008! He mentions using an object expression in order to provide an implementation of `IBuildEngine`, which I'm getting away with by setting the `Silent` property on the `ResolveAssemblyReference` task, meaning that it never attempts to obtain the logging context from the build engine.

Another gotcha here is the fact that versions of various interfaces including `IBuildEngine` and `ITaskItem` exist in both the 2.0.0.0 and 3.5.0.0 versions of the `Microsoft.Build.Framework` assembly. Be careful that you reference the correct version for the task that you're calling. Normally it will be the 2.0 versions, in which case you'll need to use explicit file references in your script, adding the directory to the library include path:

    
    
    #I @"C:\winnt\Microsoft.NET\Framework\v2.0.50727\"
    #r @"Microsoft.Build.Framework.dll"
    


Rather than partially qualified assembly references:

    
    
    #r @"Microsoft.Build.Framework"
    


As the latter will resolve to the latest installed version of the assembly, 3.5, which is not what you want. Look out for errors like:



<blockquote>
System.InvalidCastException: Unable to cast object of type 'Microsoft.Build.Utilities.TaskItem' to type 'Microsoft.Build.Framework.ITaskItem'.
</blockquote>







## Alternatives


[caption id="attachment_777" align="alignleft" width="300" caption="The Tools|Options MSBuild verbosity setting"][![The Tools|Options MSBuild verbosity setting](build/gatsby/www.ianvoyce.com/assets/2010-03-31-calling-msbuild-tasks-with-fsharp_msbuild_verbosity-300x174.png)](http://www.ianvoyce.com/wp-content/uploads/2010/03/msbuild_verbosity.png)[/caption]Although this was a useful experiment, I discovered that it's often much easier to just increase the MSBuild verbosity and wade through the reams of output to find what you're after (but then [the Yak will remain hairy](http://en.wiktionary.org/wiki/yak_shaving)). There's an option in Tools|Options|Projects and Solutions|Build and Run to change the output level from Silent to Diagnostic. Beware: you can easily get megabytes of text output from diagnostic mode!

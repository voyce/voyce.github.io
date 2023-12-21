---
date: 2009-03-30 10:17:52+00:00
description: ''
featuredImage: ''
slug: /verifying-dynamic-il
template: blog-post
title: Verifying dynamically generated IL
categories:
- .NET
- F#
- Software Development
tags:
- .NET
- CLR
- emit
- IL
- peverify
- reflection
---

It's safe to assume that when you use the C#, F# or (heaven forfend) VB.NET compilers, the IL generated for you will be correct. But, if you're using Reflection.Emit to generate code "by hand" in a dynamic method or assembly it can be difficult to identify problems with the IL you emit. In the majority of cases the runtime will simply throw an InvalidProgramException. This is of course, exactly as you'd expect, as the JIT compiler (which generates architecture-specific machine code from the IL) is intended to be highly performant, rather than robust to errors which should've been dealt with earlier in the tool chain.

So what tools can you use to troubleshoot problems with dynamic IL? In a word: peverify.
<!-- more -->
**What is peverify?**
peverify (where PE stands for portable executable, the file format used by Windows executables) takes a .NET assembly and validates the metadata - the type structure - and IL - the instructions, using a combination of rules and static analysis.

As the MSDN pages states it's intended for use by "compiler writers and script engine developers", but with a few small modifications to your code you can also use it with Reflection.Emit.

Here's some example code that uses F# to emit a simple type (deriving from MarshalByRefObject) to wrap another object. The intention is that the wrapper type implements the same interfaces as the underlying object, and delegates calls to it. The key part of the code is where we emit the body of each function.




    
    <span style="color: blue;">let</span> wrapObject (obj:obj) =



    
        <span style="color: blue;">let</span> name = <span style="color: maroon;">"MyAssembly"</span>



    
        <span style="color: blue;">let</span> filename = name + <span style="color: maroon;">".dll"</span>



    
        <span style="color: blue;">let</span> ab = AppDomain.CurrentDomain.DefineDynamicAssembly(AssemblyName(name), AssemblyBuilderAccess.RunAndSave)



    
        <span style="color: blue;">let</span> modb = ab.DefineDynamicModule(filename, filename)



    
        <span style="color: blue;">let</span> tb = modb.DefineType(<span style="color: maroon;">"MyType"</span>, TypeAttributes.Class, typeof<MarshalByRefObject>)



    
        <span style="color: blue;">let</span> objField : FieldBuilder = tb.DefineField(<span style="color: maroon;">"_underlyingObject"</span>, typeof<obj>, FieldAttributes.Public)



    
        obj.GetType().GetInterfaces() 



    
        |> Array.iter (<span style="color: blue;">fun</span> itf <span style="color: blue;">-></span>



    
            tb.AddInterfaceImplementation(itf)



    
            itf.GetMethods()



    
            |> Array.iter (<span style="color: blue;">fun</span> m <span style="color: blue;">-></span>



    
                <span style="color: blue;">let</span> ptypes = m.GetParameters() |> Array.map (<span style="color: blue;">fun</span> p <span style="color: blue;">-></span> p.ParameterType)



    
                <span style="color: blue;">let</span> mb = tb.DefineMethod(m.Name, MethodAttributes.Public|||MethodAttributes.Virtual, m.ReturnType, ptypes)



    
                <span style="color: blue;">let</span> il = mb.GetILGenerator()



    
                <span style="color: green;">// load object reference from field</span>



    
                il.Emit(OpCodes.Ldarg_0)



    
                il.Emit(OpCodes.Ldfld, objField)



    
                <span style="color: green;">// cast it to appropriate type</span>



    
                il.Emit(OpCodes.Castclass, itf)



    
                <span style="color: green;">// push all args, verbatim</span>



    
                m.GetParameters() |> Array.iteri (<span style="color: blue;">fun</span> n p <span style="color: blue;">-></span> il.Emit(OpCodes.Ldarg,n+1))



    
                <span style="color: green;">// call method on underlying objects</span>



    
                il.Emit(OpCodes.Callvirt, m)



    
                il.Emit(OpCodes.Ret)



    
            ))



    
        <span style="color: blue;">let</span> typ = tb.CreateType()



    
        ab.Save(filename)



    
        <span style="color: blue;">let</span> o = typ.GetConstructor([||]).Invoke([||])



    
        typ.GetField(<span style="color: maroon;">"_underlyingObject"</span>).SetValue(o, obj)



    
        o






Now, in my first version of the code, I forgot that you need to load "this" onto the stack before loading a field, so the code looked like this:





let il = mb.GetILGenerator()




// il.Emit(OpCodes.Ldarg_0) Oops, the forgot to load "this" onto stack




il.Emit(OpCodes.Ldfld, objField)




il.Emit(OpCodes.Castclass, itf)






I used the following code to create an instance of a type implementing IFoo, wrap it and invoke the function:




    
    <span style="color: blue;">do</span>



    
        <span style="color: blue;">let</span> o = 



    
            { <span style="color: blue;">new</span> MyInterfaces.IFoo <span style="color: blue;">with</span>



    
                <span style="color: blue;">member</span> this.Bar(a) = a + <span style="color: maroon;">"!"</span> }



    
        <span style="color: blue;">let</span> wrappedObj = wrap o



    
        wrappedObj.GetType().InvokeMember(<span style="color: maroon;">"Bar"</span>, BindingFlags.Instance|||BindingFlags.Public|||BindingFlags.InvokeMethod, <span style="color: blue;">null</span>, wrappedObj, [|box <span style="color: maroon;">"Hello"</span>|])



    
        ()






As you'd expect the function generated the expected InvalidProgramException:
`
System.Reflection.TargetInvocationException: Exception has been thrown by the target of an invocation. ---> **System.InvalidProgramException: Common Language Runtime detected an invalid program.**
   at MyType.Bar(String )
   --- End of inner exception stack trace ---
   at System.RuntimeMethodHandle._InvokeMethodFast(Object target, Object[] arguments, SignatureStruct& sig, MethodAttributes methodAttributes, RuntimeTypeHandle typeOwner)
   at System.RuntimeMethodHandle.InvokeMethodFast(Object target, Object[] arguments, Signature sig, MethodAttributes methodAttributes, RuntimeTypeHandle typeOwner)
`

In order to use peverify with your code, the first thing you need to do is save the assembly to disk. This involves making a couple of changes to the code, because normally you'd do everything in memory and avoid the cost of the disk access. First, change Run to RunAndSave in the call to define the assembly:






let ab = AppDomain.CurrentDomain.DefineDynamicAssembly(AssemblyName(name), AssemblyBuilderAccess.**RunAndSave**)





And ensure you include the filename when you create the module:





let modb = ab.DefineDynamicModule(filename**, filename**)






Now, you should be able to save the assembly to disk by adding a call after you're finished building the type:





ab.Save(filename)






Be aware that you can't specify a path in the call to AssemblyBuilder.Save. This is presumably a security restriction; keeping generated binaries within the application's directory tree and preventing people creating arbitrary binaries all over the file system. If you're not running from an obvious "top level" application, e.g. you're using a scripting environment like F# interactive, you may find the location of the file odd; when using FSI from within Visual Studio, the binary is created in "C:Program FilesMicrosoft Visual Studio 9.0Common7IDE".

Now we can run peverify on the DLL and we get a much richer explanation of the problem:

`
Microsoft (R) .NET Framework PE Verifier.  Version  3.5.30729.1
Copyright (c) Microsoft Corporation.  All rights reserved.
``
[IL]: Error: [c:Program FilesMicrosoft Visual Studio 9.0Common7IDEMyAssembl
y.dll : MyType::Bar][offset 0x00000000] **Stack underflow**.
1 Error(s) Verifying c:Program FilesMicrosoft Visual Studio 9.0Common7IDEMy
Assembly.dll
`

From this error message it's pretty clear that the first thing we're doing (i.e. the instruction at offset 0) is underflowing the stack; calling an instruction that expects more on the stack than we've put there. That should give us a good idea of how to fix the problem.

Another error that I came across was the omission of a cast in the IL:





let il = mb.GetILGenerator()




il.Emit(OpCodes.Ldarg_0)




il.Emit(OpCodes.Ldfld, objField)




//il.Emit(OpCodes.Castclass, itf) Oops, forget to cast object to specific type






From which peverify generated the fantastically specific error message:
`
[IL]: Error: [c:Program FilesMicrosoft Visual Studio 9.0Common7IDEMyAssembl
y.dll : MyType::Bar][offset 0x0000000C][found ref 'System.Object'][expected ref
'MyInterfaces.IFoo'] Unexpected type on the stack.
`

And of course, once all your emission bugs are fixed, you should get a success report:
`
All Classes and Methods in c:Program FilesMicrosoft Visual Studio 9.0Common7
IDEMyAssembly.dll Verified.
`

The only problem with peverify as far as I can see is that there are no compiler style "error codes" to help you identify and resolve the problem. It assumes fairly intimate knowledge of the IL/CLR specification and there may be a bit of effort required to get from the error to the resolution. But one thing's for sure: it's definitely easier than trying to do it with just InvalidProgramException.

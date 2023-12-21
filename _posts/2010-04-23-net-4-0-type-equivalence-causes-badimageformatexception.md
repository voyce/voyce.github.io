---
date: 2010-04-23 10:53:33+00:00
description: ''
excerpt: Interop assemblies containing certain constructs will cause a BadImageFormatException
  in .NET 4.0
featuredImage: ''
slug: /net-4-0-type-equivalence-causes-badimageformatexception
template: blog-post
title: .NET 4.0 Type Equivalence causes BadImageFormatException
categories:
- .NET
- COM
- Debugging
- WinDbg
- Windows
tags:
- .NET
- .NET4
- CLR
- COM
- Debugging
- IL
- WinDbg
---

I recently discovered a nasty backward compatibility problem with the new type equivalence feature in .NET 4.0. Luckily it's relatively difficult to hit it if you're in a pure-C# environment, but if you happen to generate any assemblies directly using IL, you should watch out. Read on for all the gory details.
<!-- more -->


## What is .NET type equivalence?


Described at a high level [here](http://msdn.microsoft.com/en-us/library/dd997297.aspx), .NET 4.0 type equivalence essentially gives you a way of indicating that different .NET types represent the same underlying COM type and is most commonly used in COM interop scenarios. One of the reasons for its introduction is to save developers from having to ship large interop DLLs with their software, e.g. the multi-megabyte Microsoft.Office.Interop. Instead the compiler can inline the definition of any types used, and mark them appropriately as representing the original COM types. 



## The error


We noticed that whenever we built and ran an application that referenced a DLL using .NET 2.0, it worked. Doing the same thing with .NET 4.0 caused a [BadImageFormatException](http://msdn.microsoft.com/en-us/library/system.badimageformatexception.aspx).
`
Unhandled Exception: System.BadImageFormatException: Could not load file or assembly 'mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089' or one of its dependencies. An attempt was made to load a program with an incorrect format.
   at X.Main()
` 


## Let's dig!


So, the BadImageFormatException doesn't actually tell us much. Let's break out WinDbg and see what we can find. Running the faulting app we can see several C++ exceptions before the CLR exception is thrown:
`
(178c.790): C++ EH exception - code e06d7363 (first chance)
...
(178c.790): C++ EH exception - code e06d7363 (first chance)
(178c.790): CLR exception - code e0434352 (first chance)
`
I changed the exception handling settings to stop on C++ exceptions (`sxe eh`) then ran again to see where things were going wrong. It stopped here:
`
0:000> kp
ChildEBP RetAddr  
0012d15c 79084c0f KERNEL32!RaiseException+0x53
0012d194 793371be MSVCR100_CLR0400!_CxxThrowException+0x48
0012d5e4 79455cae clr!EEFileLoadException::Throw+0x1a8
0012d634 794558d2 clr!CompareTypeTokens+0x200
0012d6b0 791b5c00 clr!IsTypeDefEquivalent+0x102
0012d6d4 791b2ca8 **clr!MethodTableBuilder::CheckForTypeEquivalence**+0x94
0012d7ac 791b27c9 clr!MethodTableBuilder::BuildMethodTableThrowing+0x60d
0012d9a4 791a4578 clr!ClassLoader::CreateTypeHandleForTypeDefThrowing+0x88e
`
Interesting. Notice how the call stack contains some .NET 4.0 specific methods relating to the new type equivalence feature. We're hitting a new code path, which is consistent with the fact that running against a down-level CLR works.

After a bit more toing-and-froing, I discovered that the C++ exception is thrown when `clr!MDInternalRO::IsValidToken` returns an error. By disassembling the function we can see it's just looking at various bits in the token value, and it decides that the value passed (0x02000000) isn't valid. Looking at the output from ildasm that token doesn't appear anywhere. And if we add a dump of the value, we can see that it indeed doesn't look like the other tokens: 


    
    
    0:000> bu clr!MDInternalRO::IsValidToken "dd esp+8 L1; g"
    ...
    0012f5a8  02000001
    0012f31c  06000001
    0012f2c0  02000002
    0012f0f4  02000002
    0012ebe4  01000001
    0012e944  23000001
    ...
    0012d5f4  02000000
    (18ec.1ec8): C++ EH exception - code e06d7363 (first chance)
    





## What's the culprit?


So it looks pretty conclusive; the DLL contains something that the CLR isn't expecting. But what? It's time to break out the oldest tool in the troubleshooting box: the binary chop!

Eventually I got the referenced DLL down to only a single simple construct. Can you guess what it is? A global literal value. A _real_ global value, one that isn't even part of a type. Crazy huh? In IL it looks like this:
`
.field public static literal valuetype Test.MyEnum LiteralValue = int32(0x00000001)
`
It's a literal value of an enumerated type. That's important: using a value of a simple type (say int32) does not provoke the error.

Now, I wasn't even sure that this is a valid IL construct, but according to the ECMA IL spec, specifically [partition II, section 15](http://jilc.sourceforge.net/ecma_p2_cil.shtml#_Toc524940530), it is:



<blockquote>The CLI also supports global fields, which are fields declared outside of any type definition. Global fields shall be static.</blockquote>



So it looks like we're not doing anything illegal, backed up by the fact that the .NET 2.0 CLR can make use of it without a problem.

Interestingly, there's another aspect that influences whether this code path is hit. As mentioned above, type equivalence is intended for use with interop libraries. As such, it only kicks in if your referenced assembly is marked with the PrimaryInteropAssembly attribute, e.g.:

`
  .custom instance void [mscorlib]System.Runtime.InteropServices.PrimaryInteropAssemblyAttribute::.ctor(int32,int32) = ( 01 00 01 00 00 00 00 00 00 00 00 00 ) 
`



## The Fix?


The issue is currently with Microsoft product support. Let's see what they come up with; is it too esoteric for a hotfix...?



## The Repro


Here's some code and instructions on how to repro the problem.





  1. Build the IL into a DLL using ilasm. 
`"c:\WINNT\Microsoft.NET\Framework\v2.0.50727\ilasm.exe" /dll Test.il /output=Test.dll`



  2. Build the application into a .NET 4.0 EXE that references the DLL
`"c:\winnt\Microsoft.NET\Framework\v4.0.30319\csc.exe" TestConsumer.cs /reference:Test.dll`



  3. Run the resulting `TestConsumer.exe` application and you'll get the exception


**Test.il**
`
.assembly extern mscorlib
{
  .publickeytoken = (B7 7A 5C 56 19 34 E0 89 ) 
  .ver 2:0:0:0
}
.assembly Test
{
  .custom instance void [mscorlib]System.Runtime.InteropServices.PrimaryInteropAssemblyAttribute::.ctor(int32,int32) = ( 01 00 01 00 00 00 00 00 00 00 00 00 ) 
  .hash algorithm 0x00008004
  .ver 1:0:0:0
}
.module Test.dll
.imagebase 0x00400000
.file alignment 0x00000200
.stackreserve 0x00100000
.subsystem 0x0003  
.corflags 0x00000001 

.field public static literal valuetype Test.MyEnum LiteralValue = int32(0x00000001)

.class public auto ansi sealed Test.MyEnum
       extends [mscorlib]System.Enum
{
  .field public specialname rtspecialname int32 value__
  .field public static literal valuetype Test.MyEnum Zero = int32(0x00000000)
  .field public static literal valuetype Test.MyEnum One = int32(0x00000001)
}
`
**TestConsumer.cs**

    
    
    class X
    {
        static void Main()
        {
            var v = Test.MyEnum.Zero;
        }
    }
    

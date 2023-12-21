---
date: 2009-10-23 19:51:27+00:00
description: ''
excerpt: 'Lambdas in C++ and F#: compare and contrast.'
featuredImage: build/gatsby/www.ianvoyce.com/assets/2009-10-23-comparing-lambdas-in-c-and-f_printf_tooltip-150x64.png
slug: /comparing-lambdas-in-c-and-f
template: blog-post
title: Comparing lambdas in C++ and F#
categories:
- F#
- Software Development
- Visual Studio
tags:
- c++
- F#
---

The other day someone pointed me to the new [MSDN article on C++ lambdas](http://msdn.microsoft.com/en-us/library/dd293608(VS.100).aspx) in Visual Studio 2010. Interesting. C++0x has been a long time coming, but in my opinion anything that makes C++ usable at a higher level of abstraction is a good thing. There's nothing wrong with a little abstraction, as long as you can get to the fundamentals if you need to. Otherwise you wouldn't be using C++, would you?

But one of the things I couldn't get over from that article is the verbosity. I've just got too used to the succintness of F#. I know this is only example code, so it's intended to demonstrate usage, but the whole point of lambdas is to reduce boilerplate and make the intent of your code more obvious. So, let's compare the C++0x implementation with the F# version.
<!-- more -->
Here's the C++ version, with the comments removed (including the "increment the counter" comment from the original article... Oh, please):


    
    
    #include <algorithm>
    #include <iostream>
    #include <vector>
    using namespace std;
    
    int main()
    {
       vector<int> v;
       for (int i = 0; i < 10; ++i)
       {
          v.push_back(i);
       }
    
       int evenCount = 0;
       for_each(v.begin(), v.end(), [&evenCount] (int n) {
          cout << n;
    
          if (n % 2 == 0)
          {
             cout << " is even " << endl;
    
             evenCount++;
          }
          else
          {
             cout << " is odd " << endl;
          }
       });
    
       cout << "There are " << evenCount << " even numbers in the vector." << endl;
    }
    


As well as the verbosity, notice the weird syntax used to capture the `evenCount` variable.

Without further ado, here's an F# version - not a definitive one necessarily - that returns the value rather than doing nasty side-effecting IO by printing it out to the screen:

    
    
    seq { 0..9 } |> Seq.filter (fun e -> e % 2 = 0) |> Seq.length
    


If you want to make it a bit more like the original code, you can add back the  `printf`s, and lay it out a bit more logically:

    
    
    let count =
        seq { 0..9 }
        |> Seq.fold (fun count e ->
            match e % 2 with
            | 0 -> printf "%d is even\n" e; count + 1
            | _ -> printf "%d is odd\n" e; count) 0
    printf "There are %d even numbers" count
    


I guess the comparison isn't really a fair one. The fundamental difference between C++ and F# lambdas is that they're an afterthought in C++. In F# they're a core, fundamental part of the language, and have been from day one. Functions are created, passed and bound as a matter of course.

The C++ heritage becomes obvious when you look at the finer points of its lambda syntax. For instance, having to specify whether to capture variables by reference or by value, in the "capture clause" or lambda-introducer. This is the type of distinction that both .NET programmers and functional programmers are not used to having to make.



## Other nice F# features


Although not directly related to lambdas, there are some other F# language features that help make code more concise and understandable:




  1. ### Sequence expressions


F# has an extremely terse syntax for generating and manipulating sequences - collections that implement IEnumerable. This means you can easily generate collections and iterate over them, without the potential for off-by-one errors, the scourge of `for`

  2. ### Fewer brackets


In F#, whitespace is significant. It takes some time to get used to, but it ultimately results in cleaner code. It's been interesting to see the use of `#light` gradually become more and more standard as F# has neared an official release. Now it's implicit in all code. Try it, you'll like it.



  3. ### Type-safe printf


For some reason I always preferred the C style `printf` string formatting to C++s `cout` stream based version. There's something more readable about printf, but it suffers from a terrible lack of compile time type safety. This can be disastrous if, for instance, you happen to wrongly type the insert in an error message or other code path that's rarely hit. The one time you need it, the error handling itself generates an error! Ouch.

[![printf_tooltip](http://www.ianvoyce.com/wp-content/uploads/2009/10/printf_tooltip-150x64.png)](http://72.47.193.211/wp-content/uploads/2009/10/printf_tooltip.png)
Luckily, F# combines the best of both worlds with a compile-time type-safe printf! At first glance this can seem like some kind of magic to C++ developers (well, it did for me).



Now that F# is officially "out there" and available as a first class citizen in Visual Studio 2010, you should really take a look. I'm increasingly convinced that the cost of using C++, not necessarily the [raw, low-level cost](http://www.rachelslabnotes.com/2009/10/the-hidden-cost-of-c/), but the business costs involved with writing performant, maintainable, manageable code in C++, is getting too much to bear. But that's probably the basis for another post.

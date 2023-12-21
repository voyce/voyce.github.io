---
date: 2008-06-20 07:09:10+00:00
description: ''
featuredImage: ''
slug: /programmatically-checking-memory-usage
template: blog-post
title: Programmatically checking memory usage
categories:
- Software Development
- Windows
tags:
- heap
- pdh
- perfmon
- private bytes
- win32
---

One of the things that's useful in a pre-release check is do a regression test on the memory usage of your unmanaged functions. This should help to ensure that the fantastic new data structure you introduced doesn't cost _too_ much in additional storage for the order-of-magnitude performance improvement you were boasting about.

Like most of my posts, this assumes that it's not feasible to go through all your source code, and say, replace all instances of new with a version that tracks usage (the approach used by the debug CRT). As well as being logistically infeasible, this also tends to miss allocations that don't go via new, for example, direct calls to HeapAlloc.

<!-- more -->In the past, I've seen some code trying to use the [Win32 heap functions](http://msdn.microsoft.com/en-us/library/aa366781(VS.85).aspx) to try and find out the amount of memory allocated by the process. It used GetProcessHeaps, HeapWalk and HeapSize to sum all the block sizes and get an overall memory in use figure, but in my experience it was extremely slow and unreliable.

What was really required was something that gave a figure similar to the "private bytes" counter in perfmon. If you didn't know, this is the counter you need to be watching if you're looking for memory leaks in a process. For goodness sake don't use the "Mem Usage" column in Task Manager; this is in fact (almost) the working set size and it doesn't correlate exactly with memory explicitly allocated by the process. It includes additional things including space occupied by the loaded DLLs. Also, the working set will shrink if the app is paged out, although it still has the memory allocated. To see an example of this in action, open Excel and a large spreadsheet, calc it, look in Task manager and you'll see a large number (if not, you're obviously not looking at a _real_ spreadsheet). Then minimise the Excel window. You'll see the mem usage value plummet as the working set is "trimmed" - probably by a call to [SetProcessWorkingSetSize](http://msdn.microsoft.com/en-us/library/ms686234(VS.85).aspx). The OS does this because it expects the app won't be being used, so it makes sense to free up physical memory for use by other processes.

So essentially what I want to do is get the perfmon "private bytes" value programmatically as my app is running, and this can be achieved using the Performance Data Helper (PDH) library. It provides an API to access the performance counters in a similar way to the perfmon GUI.

It uses the concept of "queries"; you create a query, add a counter to it, then collect the query data as required (not forgetting to remove the counter and close the query when you're done).

The first thing to do is open the query:





    PDH_STATUS status = PdhOpenQuery(NULL, 0, &hquery);




    if (status != ERROR_SUCCESS)




        return status;






 

Then add the required counters (this code assumes you're looking at a process on the current machine):





    status = PdhAddCounter(hquery, _T("\\\\.\\Process(processname)\\Private Bytes"), 0, &hcounter);




    if (status != ERROR_SUCCESS)




    {




        PdhCloseQuery(hquery);




        return status;




    }






At this point you're ready to start polling for updates. At periodic intervals you can collect the query data and do with it what you will:





    PDH_STATUS status = PdhCollectQueryData(hquery);




    if (status == ERROR_SUCCESS)




    {




        PDH_RAW_COUNTER value;




        DWORD dwType;




        status = PdhGetRawCounterValue(hcounter, &dwType, &value);




        if (status == ERROR_SUCCESS)




        {




            printf("%lld %lld %s\n", value.TimeStamp, value.FirstValue, sz);




        }




    }






Luckily for the Private Bytes counter we've got the simplest type of counter to 'decode'; a raw counter value, essentially just a number. We don't need to do any further manipulation on it to get the information we need, like having to divide by some frequency.

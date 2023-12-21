---
date: 2008-06-11 08:20:46+00:00
description: ''
featuredImage: ''
slug: /creating-a-featherweight-debugger
template: blog-post
title: Creating a featherweight debugger
categories:
- Debugging
- Software Development
- WinDbg
- Windows
---

What do I mean by "featherweight debugger"? I mean implementing just enough of the debugging framework to get what we need from the debuggee and nothing more.

The problem I was trying to solve was how to get more information from first chance exceptions. We have a great deal of library code that uses catch(...) blocks to prevent any exceptions - even structured ones like access violations - from escaping into the application. This is pretty important when your code is running inside an Excel session that someone may've been working on for hours (and which could take minutes just to re-calculate in the event of having to re-open it). Unfortunately, because the catch handlers are built into all the code, if you discover something causing access violations, say, there's no single place that you can change to get more diagnostic information.

So, with source level changes ruled out, how else could you catch exceptions?

Well, the obvious thing that does this already is the debugger, so I decided to look at the [debugging API](http://msdn.microsoft.com/en-us/library/ms679300(VS.85).aspx), and see if it was possible to use that. It turns out that the implementation is pretty simple; essentially the debugger simply waits on an event in a loop, receiving further specific information about the event when it's fired.


```
    DEBUG_EVENT DebugEv;                   // debugging event information 
    DWORD dwContinueStatus = DBG_CONTINUE; // exception continuation 

    while (true)
    {
        WaitForDebugEvent(&DebugEv, INFINITE);
        switch (DebugEv.dwDebugEventCode)
        {
            // Do something with event
            // Set dwContinueStatus to tell debuggee what to do 
        }

        // Resume executing the thread that reported the debugging event. 
        ContinueDebugEvent(DebugEv.dwProcessId,
                           DebugEv.dwThreadId,
                           dwContinueStatus);
    }

```




There are several 'informational' events, DLLs being loaded and unloaded etc, and, more interestingly, there's one called EXCEPTION_ACCESS_VIOLATION. Voila!

When this event is received, we can use the additional information we're passed, along with some thread context data, to create a minidump. This is an extremely useful mechanism for snapshotting the process in a rich enough way to provide full post-mortem debugging abilities.

Here's some sample code to create a minidump using the with enough information to be able to (amongst other things):



	
  * Get stack traces

	
  * Get memory usage information (using !address -summary)


```
DWORD CreateMiniDump(DEBUG_EVENT ev)
{
    HANDLE hproc = OpenProcess(PROCESS_ALL_ACCESS, FALSE, ev.dwProcessId);

    if (!hproc)
        return ReportError(_T("Failed to open process"),GetLastError());

    EXCEPTION_POINTERS ep;
    ep.ExceptionRecord = &ev.u.Exception.ExceptionRecord;

    HANDLE hthread = OpenThread(THREAD_ALL_ACCESS, FALSE, ev.dwThreadId);

    if (!hthread)
    {
        DWORD dw = GetLastError();
        CloseHandle(hproc);

        return ReportError(_T("Failed to open thread"),dw);
    }

   CONTEXT ctxt;
   ZeroMemory(&ctxt, sizeof(ctxt));

    GetThreadContext(hthread, &ctxt);
    ep.ContextRecord = &ctxt;

    TCHAR sz[MAX_PATH];
    _stprintf_s(sz, _countof(sz), _T("c:\\dump_%ld_%ld_%ld.dmp"), ev.dwProcessId, ev.dwThreadId, nDumps++);

    HANDLE hfile = CreateFile(sz, GENERIC_WRITE, FILE_SHARE_READ, NULL, CREATE_ALWAYS, 0, NULL);

    if (!hfile)
    {
        DWORD dw = GetLastError();
        CloseHandle(hproc);

        return ReportError(_T("Failed to create output file"),dw);
    }

    MINIDUMP_EXCEPTION_INFORMATION info;
    info.ClientPointers = FALSE;
    info.ThreadId = ev.dwThreadId;
    info.ExceptionPointers = &ep;
    if (!MiniDumpWriteDump(hproc, ev.dwProcessId, hfile,
        MiniDumpWithFullMemory,
        &info,
        NULL,
        NULL))
    {
        DWORD dw = GetLastError();
        CloseHandle(hfile);
        CloseHandle(hproc);
        return ReportError(_T("Failed to write mini dump"),dw);
    }
    CloseHandle(hfile);
    CloseHandle(hproc);

    _tprintf(_T(" Minidump written to %s\n"), sz);
    return ERROR_SUCCESS;
}
```

Caveat: Example code only, please excuse the hard-coded path!

The debugger loop is then wrapped up as a stand-alone application that can be passed the PID of a process to debug. Assuming we've got sufficient permissions, our debugger can attach to that process using DebugActiveProcess(pid).

So, here we have a relatively lightweight way of 'watching' another process and grabbing lots of potentially useful diagnostic information in the event that something untoward happens. Of course when using this in the wild you'd also need some means of tidying up the generated files, and potentially some means of logging that the event had occurred, but these are relatively straightforward to implement.

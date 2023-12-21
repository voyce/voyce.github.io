---
date: 2007-08-13 22:46:52+00:00
description: ''
featuredImage: ''
slug: /process-and-thread-tokens
template: blog-post
title: Process and thread tokens
categories:
- Software Development
---

I’ve recently been trying to work out what was going on with some ASP.NET/COM interop issues at work. It turned out to be due to the differences between [OpenProcessToken](http://msdn2.microsoft.com/en-us/library/aa379295.aspx) and [OpenThreadToken](http://msdn2.microsoft.com/en-us/library/aa379296.aspx).


It might sound obvious in hindsight, but I haven’t worked a great deal with the security model within Windows, so I’m not particularly au fait with it all. Plus, given that it was a DLL, called from COM, called from IIS, it wasn’t particularly easy to debug.

The system I work on uses an access control mechanism whereby users have to be members of a certain NT domain group in order to use it. At certain points in the process, the security DLL checks for this by getting the current token, using [GetTokenInformation ](http://msdn2.microsoft.com/en-us/library/aa446671.aspx)and iterating over the group SIDs it contains (it was written before [CheckTokenMembership](http://msdn2.microsoft.com/en-us/library/aa376389.aspx) was available).

The trouble is, when running under IIS and ASP.NET, the check was always failing. Even though the appropriate user identity was being passed through, by impersonation, from the client, it wasn’t working. Hmmm.

It turned out that the validation code was using OpenProcessToken, but of course, the impersonation happens at thread level. You can impersonate as much as you want, but the process access token always contains the original token (for the Network user in my case), not the one with the groups for the user you’re interested in.

By changing the code to use OpenThreadToken and passing FALSE for the OpenAsSelf parameter, you can get the properly impersonated access token. Ahhh.

---
date: 2007-08-15 22:25:41+00:00
description: ''
featuredImage: ''
slug: /getting-com-registration-data-with-the-activation-context-api
template: blog-post
title: Getting COM registration data with the activation context API
categories:
- COM
- Software Development
---

When moving code from “traditional” registry-based COM to shiny new side-by-side, registration-free COM, there are a few places where you might need an analog for things like looking up a DLL name from a prog ID. E.g. in the registry-based world, you can go from a Prog ID for a class to it’s physical DLL filename by doing this:



	
  * Get CLSID from the ProgID

	
  * Look up filename in HKCR\CLSID\{clsguid}\InprocServer32


Now obviously, if you attempt this on a machine where the components are only being used via SxS (i.e. have never been registered) the CLSIDFromProgID step will work - OLE32.DLL, which implements this function, is aware of SxS - but the subsequent steps won’t because the information isn’t in the registry.

I guess this is really breaking because we’re taking advantage of our knowledge of COM internals, rather than going via the official APIs. Although as far as I know the “correct” way of doing the progid->(type library) DLL mapping is via [IProvideClassInfo](http://msdn2.microsoft.com/en-us/library/ms687303.aspx), but that relies on the COM objects supporting this interface, and unfortunately we have, err, several - hundred - that don’t.

So, I set about looking to see if there was a way to get this information using the activation context APIs. All the information is there in the manifest, so how do we get it - without something nasty like querying the manifest XML directly?

There are only a handful of functions in the ActCtx API, and one of them - **[**FindActCtxSectionGuid**](http://msdn2.microsoft.com/en-us/library/aa375148.aspx)** - looks relevant. By calling this with the **ACTIVATION_CONTEXT_SECTION_COM_SERVER_REDIRECTION** flag, it looks like we can get some data from the manifest based on our COM object CLSID.

Here’s the problem. The returned data from this function is a **ACTCTX_SECTION_KEYED_DATA** structure, and as far as I can tell this is essentially an opaque blob, with a couple of length indicators. I couldn’t find any documentation about what the lpData member was supposed to point to (if you know any better, please let me know)!

I decided to break out WinDbg, and see what OLE32!CLSIDFromProgID did, as I assumed that this must be doing something similar. It was! In fact, it was calling FindActCtxSectionString to map the prog ID to a CLSID, then using this in a call to FindActCtxSectionGuid. After a bit of disassembly, and some staring at the memory window in Visual Studio, I got a good enough idea of the contents to be able to figure out how the data referenced the filename:


   typedef struct tagSECTION_DATA




   {




       DWORD dwSize; // 0×78 (120) structure size?




       DWORD _2; // 0×00




       DWORD dwSectionType; // 0×04 (ACTIVATION_CONTEXT_SECTION_COM_SERVER_REDIRECTION)




       GUID clsid; // CLSID of class?




       GUID _5; // Some other GUID




       GUID _6; // CLSID of class again…?




       GUID _7; // NULL




       DWORD dwFileNameLength; // file name size in bytes




       DWORD dwFileNameSectionOffset; // file name offset into data.lpSectionBase




       DWORD dwProgIDLength; // progid size in bytes




       DWORD dwProgIDOffset; // offset from start of this structure to progid (0×78)




       BYTE _8[28]; //Unknown




       // Prog ID string follows




   } SECTION_DATA;


So now you can cast the data member to this structure and easily extract the filename, voila!


   ACTCTX_SECTION_KEYED_DATA data;




   data.cbSize = sizeof(data);




 




   if (!FindActCtxSectionGuid(FIND_ACTCTX_SECTION_KEY_RETURN_HACTCTX,




       NULL,




       ACTIVATION_CONTEXT_SECTION_COM_SERVER_REDIRECTION,




       &guid,




       &data))




   {




       return GetLastError();




   }




   if(data.ulDataFormatVersion == 1) // Fail-safe in case internal format changes…?




   {




       // Cast returned data to our structure type




       SECTION_DATA *pdata = static_cast<SECTION_DATA *>(data.lpData);




       // DLL filename can be found in the section base data at specified offset




       std::wstring filename(




           reinterpret_cast<wchar_t *>( ((BYTE *)data.lpSectionBase + pdata->dwFileNameSectionOffset) ));




   }


So the SxS compliant version of the CLSID to filename/typelibrary is:



	
  * Get CLSID from the ProgID

	
  * Look up GUID in current activation context using FindActCtxSectionGuid

	
  * Decipher returned data to get filename


I’m sure this will break horribly when they change the internal format of the activation context data (in fact, it’s probably different now between XP and Vista - many things in the SxS world are), but hopefully we can use the ulDataFormatVersion to do some basic sanity checking.

Now if only some of those useful looking functions in sxs.dll were documented…

---
date: 2007-03-02 22:36:53+00:00
description: ''
featuredImage: ''
slug: /bug-in-_com_ptr_tquerystdinterfaces
template: blog-post
title: Bug in _com_ptr_t::QueryStdInterfaces
categories:
- COM
- Software Development
---

Just thought I’d bring people’s attention to a bug in the COM support classes that ship with Visual Studio 2005/VC8.


It’s a fairly unusual edge case (at least, if you’re not being passed VARIANTs from a script language), where the _variant_t helper class attempts to extract a standard COM interface (IUnknown or IDispatch) from a VARIANT that’s being passed “byref”, e.g. has a type or’ed with VT_BYREF.

In this case the code in comip.h uses VariantCopy to “derefence” the pointer - convert it from, say, VT_DISPATCH|VT_BYREF to VT_DISPATCH - but if this succeeds, it then proceeds to use the wrong local variable, varSrc, rather than varDest that it’s just converted. The effect is that it causes an access violation.

Here’s the code snippet in question:







   // Try to extract either IDispatch* or an IUnknown* from




   // the VARIANT




   //




   HRESULT QueryStdInterfaces(const _variant_t& varSrc) throw()




   {




       if (V_VT(&varSrc) == VT_DISPATCH) {




           return _QueryInterface(V_DISPATCH(&varSrc));




       }




 




       if (V_VT(&varSrc) == VT_UNKNOWN) {




           return _QueryInterface(V_UNKNOWN(&varSrc));




       }




 




       // We have something other than an IUnknown or an IDispatch.




       // Can we convert it to either one of these?




       // Try IDispatch first




       //




       VARIANT varDest;




       VariantInit(&varDest);




 




       HRESULT hr = VariantChangeType(&varDest, const_cast(static_cast<const VARIANT*>(&varSrc)), 0, VT_DISPATCH);




       if (SUCCEEDED(hr)) {




           hr = _QueryInterface(V_DISPATCH(&varSrc)); // Should be &varDest




       }




 




       if (hr == E_NOINTERFACE) {




           // That failed … so try IUnknown




           //




           VariantInit(&varDest);




           hr = VariantChangeType(&varDest, const_cast(static_cast<const VARIANT*>(&varSrc)), 0, VT_UNKNOWN);




           if (SUCCEEDED(hr)) {




               hr = _QueryInterface(V_UNKNOWN(&varSrc)); // Should be &varDest




           }




       }




 




       VariantClear(&varDest);




       return hr;




   }




Looks like it’s the same in Visual Studio 2005 SP1, but Microsoft are aware of the problem, and hopefully there’ll be a patch soon.

Hope this saves you some time!

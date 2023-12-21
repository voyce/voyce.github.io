---
date: 2008-12-01 15:48:33+00:00
description: ''
featuredImage: ''
slug: /getting-net-type-information-in-the-unmanaged-world
template: blog-post
title: Getting .NET type information in the unmanaged world
categories:
- .NET
- COM
- Debugging
- Software Development
tags:
- ccw
- COM
- IDispatch
- invoke
- type
- unmanaged
---

One of the tools that I write and maintain displays type information for COM objects hidden behind "handles" in Excel spreadsheets. The underlying objects can either support an interface that allows them to be richly rendered to XML, or the viewer will fall-back to using metadata and displaying the supported interfaces and their properties and methods. It will also invoke parameterless property getters - making the assumption that doing so won't change the state of the object - and display the returned value. This is a useful way of getting some visibility on otherwise completely opaque values.

In order to obtain the type information about the COM objects, the tool uses type libraries, and the associated [ITypeLib ](http://msdn.microsoft.com/en-us/library/ms221549.aspx)and [ITypeInfo](http://msdn.microsoft.com/en-us/library/ms221696.aspx) interfaces, which, with a little effort, can be used to iterate over all the coclasses, interfaces and functions in the library. But the difficulty lies in obtaining a type library when all you're given is an already-instantiated object. In theory, COM allows you to know no more about an object than what interfaces it supports. But in practice, there are a variety of ways you can circumvent this and get to the type information.

For unmanaged COM objects you can use the information in the registry (or SxS configuration) and obtain the server (DLL) that contain a TLB embedded as a resource, or the type library filename itself. I won't go into that now, there's plenty of information about the location of these common registry keys elsewhere on the internet.

But for managed COM objects - well, COM callable wrappers (CCWs) - you have a different problem: registry scraping will never work and there may not even be an associated type library. The InprocServer32 registry entry always points to mscoree.dll, which obviously doesn't have an embedded type library, and unless you've registered the assembly with /tlb (which is a pain) then you won't have entries under HKEY_CLASSES_ROOT\Typelib and a TLB file to load.

So, if you're in the unmanaged world, and all you've got is a pointer to a live CCW, what can you do?

Well, the easiest thing is to use [IProvideClassInfo](http://msdn.microsoft.com/en-us/library/ms687303(VS.85).aspx). This is supported by all CCWs, and provides a way to get an auto-generated (by the CLR) ITypeInfo implementation for the managed class. In fact, this is what I actually used to implement the solution eventually, but along the way I discovered some other interesting aspects of the CCW.

There is another interface that it supports: _Object, the unmanaged version of System.Object, which supports basic .NET functionality such as ToString and GetType. I couldn't find it declared anywhere in the Platform or .NET SDK headers, so I put together a version that I could use from C++:





struct __declspec(uuid("{65074F7F-63C0-304E-AF0A-D51741CB4A8D}")) Object : public IDispatch




{




public:




// We don't actually call these methods, doing so seems to return




// COR_E_INVALIDOPERATION. Instead we just use the IDispatch::Invoke




// and use the DISPID of the methods.




virtual HRESULT STDMETHODCALLTYPE ToString(BSTR *) = 0;




virtual HRESULT STDMETHODCALLTYPE Equals(VARIANT, VARIANT_BOOL *) = 0;




virtual HRESULT STDMETHODCALLTYPE GetHashCode(long *) = 0;




virtual HRESULT STDMETHODCALLTYPE GetType(mscorlib::_Type **) = 0;




};






Despite the presence of the virtual functions in this "interface", we're not actually going to call them. Instead we'll call through the IDispatch that it derives from. It may be possible to use them directly, but see the comment describing what happens when I tried it. Calling via IDispatch may seem slightly odd, because the object itself claims not to support it (QueryInteface returns E_NOINTERFACE).

The methods on the _Object interface have well-known DISPIDs:
<table border="0" >
<tbody >
<tr >

<td >ToString
</td>

<td >0x00000000
</td>
</tr>
<tr >

<td >Equals
</td>

<td >0x60020001
</td>
</tr>
<tr >

<td >GetHashCode
</td>

<td >0x60020002
</td>
</tr>
<tr >

<td >GetType
</td>

<td >0x60020003
</td>
</tr>
</tbody></table>
So we can use that to invoke the GetType method:





DISPPARAMS parms;




parms.cArgs = 0;




parms.cNamedArgs = 0;




_variant_t vType;




hr = pObject->Invoke(0x60020003, IID_NULL, 0, DISPATCH_METHOD, &parms, &vType, NULL, NULL);






And we get back a [_Type](http://msdn.microsoft.com/en-us/library/system.runtime.interopservices._type.aspx) interface that allows us to navigate around the type information in the same way as we can with System.Type! Just #import mscorlib.tlb and you get all the interfaces you need to e.g. iterate over all the interfaces implemented by a type, and invoke a function on them:





#import <mscorlib.tlb> rename("ReportEvent","xReportEvent")




...




mscorlib::_TypePtr t(V_UNKNOWN(&vType));




CComSafeArray<LPUNKNOWN> saInterfaces(t->GetInterfaces());




...




mscorlib::_TypePtr tInterface((LPUNKNOWN)saInterfaces.GetAt(n));




...




result = tInterface->InvokeMember(_bstr_t("Function"),




(mscorlib::BindingFlags)




(mscorlib::BindingFlags_GetProperty +




mscorlib::BindingFlags_InvokeMethod +




mscorlib::BindingFlags_Public +




mscorlib::BindingFlags_Instance +




mscorlib::BindingFlags_IgnoreCase),




NULL, _variant_t(punk), NULL, NULL, NULL, NULL);






So this turns out to be quite nice: you can get rich managed type information even if you're running in the unmanaged world.

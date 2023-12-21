---
date: 2007-04-19 22:30:43+00:00
description: ''
featuredImage: ''
slug: /modular-excel-spreadsheets
template: blog-post
title: Modular Excel Spreadsheets
categories:
- Excel
- Software Development
---

Where I work there is no escape from the clutches of Excel. It gets everywhere. Everyone uses it for everything.One of the projects I’ve been responsible for is an attempt to modularise Excel workbooks in order to make them easier to create and manage. This is done by creating a manifest for each “component” (a worksheet, and optionally some VBA code-behind) that expresses the inputs that it consumes, and the outputs it provides. Then it’s simply a matter of recursively creating and joining these components together until we get a complete workbook.It turns out that making the abstraction that a component is nothing more than a set of inputs and outputs, and having the joining framework know nothing about the functionality of each component, means we can easily apply it across a wide range of applications. Generally it’s used by structurers to construct quite complex spreadsheets for valuing derivatives on one or many underlying assets, but it can also be used to construct “trading book” type spreadsheets that download a set of trade data and value them, automatically creating the market data components required to do the valuation.

The framework itself is written as a COM server that lives inside the Excel process, and controls it via automation, although it also works out-of-process. It supports IDispatch, so is usable from scripting languages, too. For instance I’ve put together an HTML/JavaScript based UI that can invoke the framework directly from a web browser.

The main problem is the fact that Excel is the host application. There are lots of strangenesses around the Excel automation API, but it still does a pretty good job, in my opinion, of exposing so much functionality. The real issue is that it’s a black-box. As is the VBA engine (VBE.DLL). This can make it difficult to track down problems, especially where I’m crossing backward and forward between native C++/COM code and VBA callbacks.

Still, this is a satisfying project, because it helps to formalise people’s use of spreadsheets, without them even realising. It’s easier for end-users to drive; they select what type of product they want to value from a simple GUI and the framework does the rest. And from a development point of view it means the analytics group have fewer monolithic and difficult to maintain spreadsheets. And of course, forcing us to think about the model in component terms also means that if we should ever move away from Excel (!) we would, at least in theory, have an easier migration path.

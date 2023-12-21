---
date: 2009-07-08 22:30:17+00:00
description: ''
excerpt: It's a well known fact that the financial services industry (where I mean
  banks, hedge funds, pension providers, fund managers etc) is deeply in love with
  Excel. But what is it about the Excel ecosystem that makes it so appealing, and
  how can we move seamlessly to a more robust, maintainable platform?
featuredImage: build/gatsby/www.ianvoyce.com/assets/2009-07-08-ending-the-love-affair-with-excel_spreadsheets_mug-150x150.png
slug: /ending-the-love-affair-with-excel
template: blog-post
title: Ending the love affair with Excel
categories:
- Excel
- Finance
tags:
- Excel
- Finance
- UI
---

[caption id="attachment_214" align="alignleft" width="150" caption="I heart spreadsheets"][![I heart spreadsheets](build/gatsby/www.ianvoyce.com/assets/2009-07-08-ending-the-love-affair-with-excel_spreadsheets_mug-150x150.png)](http://72.47.193.211/wp-content/uploads/2009/07/spreadsheets_mug.png)[/caption]It's a well known fact that the financial services industry (where I mean banks, hedge funds, pension providers, fund managers etc) is deeply in love with Excel. But what is it about the Excel ecosystem that makes it so appealing?

Unfortunately, given the ever-increasing focus on accountability and scalability in these domains, Excel is also an increasingly inappropriate platform for delivery of this sort of functionality. How can organisations ease the transition away from this environment to something more easily manageable, while retaining it's positive benefits?

<!-- more -->



<blockquote>Disclaimer: These are my own views and not those of my employer. The scenarios I describe are fictitious and not based on actual practices at my place of employment. Any similarity to actual events and behaviours is coincidental.
</blockquote>



I see two main aspects of Excel's operation that prove valuable in general, and in finance in particular: visibility and immediacy.



### Visibility



From an end user's point of view, one of the most positive aspects of Excel is it's transparency. Even in an environment where complex processing goes on inside a plugin - say, an Excel user defined function (UDF) packaged as an XLL - it's still only in the scope of a single function call in a single cell. You continue to get good visibility of the inputs and outputs from the function.

You can also get a good higher-level view of how the different parts of the calculation fit together. Either by inspection, editing the formula to quickly see which cells it references, or by using the formula auditing toolbar to get an overlay of dependent and precedent functions, you can see roughly how various parts of the calculation fit together.

This, combined with the logical separation of functionality into separate worksheets, gives you the impression of understandability.

The problem is that it's just that - an impression - there is no structure enforced by the worksheet separation, you're free to link between them as you see fit, and this can result in an extremely complex and virtually indecipherable "flow" of calculation within a workbook.



### Immediacy



Luckily, to offset the complexity we have another important aspect of Excel's usability: it's immediacy.

Trading users can very quickly modify calculation trees to perform ad-hoc additional calculations such as applying spreads, and to gauge the effect of market moves on their portfolio, i.e. "what-if" scenarios.

Structuring users can fairly easily - given an existing set of functional pieces - assemble a model to price a complex new trade type. Although this is a much less common use case now that the demand for getting new structured products to market quickly has essentially vanished.

As well as this type of gross modification, it's also possible to just repeatedly re-evaluate some small part of the calculation to quickly determine how it is affected by other changes.

The immediacy of the Excel environment, combined with the use of the Visual Basic for Applications (VBA) scripting language, enables both business and IT users to rapidly build out new sheets to perform ad-hoc tasks. Rapid application development, or RAD as it's known, is all well and good for performing some transient, ad-hoc calculation for limited use, but unfortunately these tools have the habit of becoming indispensable parts of an organisation's workflow. That's when you start to see serious problems with scalability and maintainability.



### The problem?



So the fundamental tension here is that exactly the same features which make Excel useful to traders, structurers, etc., also make it highly unsuitable for use in these environments. You really don't want traders to be able to subtley alter the flow of the calculation, as much as they might like to. Excel should really be a read-only view on some underlying trade data source and calculation engine, but this isn't always how it's implemented.

You also want to provide some flexibility - the ability to experiment, explore and examine - but within certain constraints and without compromising future requirements. Being able to price a trade is a good start, but you will subsequently need to book it easily, re-value it reliably and potentially risk manage a large portfolio of them.



### Moving away from Excel



Many houses provide underlying quant analytics implemented in C++, with a native C++ interface and a variety of other (sometimes auto-generated) wrappers that make it accessible in other environments. Ideally, the amount of actual functionality in these wrapper layers should be kept to an absolute minimum. It should be commodity code that can be regenerated as and when required.

However, even if this is the case, unless you have the ability in the underlying code to fan-out the processing you will still be constrained by the operating system process limits, e.g. 4gb address space.

The common alternative to hosting pricing and risk management within Excel is to write new dedicated applications to perform the calculations and (hopefully) provide a means to inspect and troubleshoot it.

Unfortunately these systems are built by teams who aren't particularly familiar with the way the front office uses Excel. It's often the case that spreadsheet development teams and systems development teams don't interact; the former tend to be based on the trading desk while the latter are in another part of the building.

So the end result can end up being systems that work in a way developers feel comfortable with, rather than the way business users would like them to work. This can make the system seem like a black box to it's users. And to make it worse these are the users who are accustomed to Excel's high levels of transparency.

Although the system may be enforcing the fact that trade value is strictly a function of the trade, market and static data, having some transparency can still be valuable in giving skilled users confidence in the underlying calculation. They can apply their not-inconsiderable intuitive abilities to quickly gauge the correctness or otherwise of the valuation. It can help the quants move away from their native Excel/Matlab-style iterative development environment. And when groups need to work together to reconcile differences, everyone can benefit.

The problem? This amount of transparency has to be built into the system or the quant library pretty much from the ground up.



### Solutions



One possible approach is to make your system a general purpose calculation engine. This has some appeal, but you have to be careful about sacrificing performance for generality. You might be tempted to try exactly matching the semantics of Excel's calculation and type system. This can be a nightmare, with all sorts of limitations and nasty edge cases. For example, the fact that errors are a simple numeric value with no facility to carry extra information is crippling, and also the lack of support for objects as first class values in the type system.

However you choose to implement your calculation engine, ensure that it's loosely coupled enough to be run out-of-process or across machine boundaries without code changes.

Here are some bullet-point takeaways for easing the transition:



	
  1. **Provide a means for users to examine intermediate calculations, and surface it in the UI.**
This will give your intermediate and end users confidence in the calculations


	
  2. **Provide a means to capture and re-play valuations, including all their inputs and outputs.**
	This will assist in reconciling valuations, and in passing snapshots around between groups.


	
  3. **Provide the ability to drive the system programatically.**
	Along with persistent valuations, this will help in regression testing




I hope this post has served to give you some insights into the potential challenges of moving users away from Excel into more robust systems, why some of those users may be reluctant to move, and how you can help to ease the transition. And of course, let me know via the comments if you've got any observations on the subject.

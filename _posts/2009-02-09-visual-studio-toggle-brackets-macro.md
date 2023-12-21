---
date: 2009-02-09 12:35:28+00:00
description: ''
featuredImage: ''
slug: /visual-studio-toggle-brackets-macro
template: blog-post
title: Visual Studio Toggle Brackets Macro
categories:
- F#
- Visual Studio
tags:
- brackets
- F#
- macro
- parentheses
- Visual Studio
---

After using a F# heavily for a while, I often found myself wanting to add brackets (or rather, parentheses) around some text. This is normally when adding a type specification to an argument in order to be able to use dot notation, e.g. going from:





let typeName t = t.Name






which causes "error FS0072: Lookup on object of indeterminate type based on information prior to this program point", to the correct:





let typeName **(t:Type)** = t.Name






(These are obviously simplistic examples!)

So I broke out the Visual Studio macro editor for the first time in a while, and put together something to toggle brackets around the currently selected text. It's naive, but, combined with Shift+Alt+Left Arrow to select the previous word, it's effective:





Public Sub AddBrackets()




Dim s As Object = DTE.ActiveWindow.Selection()




If s.Text.StartsWith("(") And s.Text.EndsWith(")") Then




s.Text = s.Text.Substring(1, s.Text.Length - 2)




Else




s.Text = "(" + s.Text + ")"




End If




End Sub






Copy this text into a module within your macro project, and assign a suitable keystroke using Tools|Customize|Keyboard.

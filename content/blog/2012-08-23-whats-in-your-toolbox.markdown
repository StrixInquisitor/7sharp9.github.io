---
author: 7sharp9
layout: post
title: "Whats in your toolbox?"
date: 2012-08-27
comments: true
categories: [programming]
tags: [F#, Clojure, Erlang, Haskell, C#, javascript, programming, tools]
description: ""
type: post
---
If I walk into my garage now and open up a toolbox, whats inside?

Here's a quick selection:

* Ball-peen hammer
* Jointer plane
* 1/2 inch mortise chisel
* Soldering iron
* Set square
* Low angle block plane
* Torx screw drivers
* Hack saw
* Monkey wrench
* Pipe cutter

Notice it doesn't just contain:  

* A sledge hammer.

Different tools have different purposes, you wouldn't use a hammer and try to cut down a tree, or use a chisel to hammer a nail.  

<!-- more -->

## Software

{{< figure src="http://farm9.staticflickr.com/8287/7817531406_9d651ccc57_d.jpg Stanley Wood Plane" >}}

In the software industry we also have access to a vast array of different tools all for different purposes:  Image editors, text editors, social media clients, email clients, it's endless!  Most of the time we pick the right tool for the job.  You wouldn't use notepad or TextMate to edit an image file, although you could, you just wouldn't.  

## Programming 
The same can be said about programming languages.  Programming languages fall into different styles, no single programming style is best suited to solve every problem.  

There are four main styles or paradigms:

* [Imperative][imperativeprogramming]  - The imperative style spells out the problem in detail, the programmer explicitly defined the exacting steps the program must follow.  
* [Functional][functionalprogramming] - Functional programming favors immutability and functional composition, the programmer thinks about the problem as sequence of stateless function evaluations.  
* [Logic][logicprogramming] - based on the idea of using logical sentences to represent programs and to perform computations.  
* [Object-oriented][objectorientedprogramming] - The programmer thinks about the problem as a collection of interacting objects.  

Some languages are said to be multi-paradigm, supposedly allowing you to use you the right tool for the job.  This only helps to a certain extent as most languages embrace a single paradigm more than others.  In addition, there are also classes of problems that cannot be easily solved simply with one of the main paradigms.  Problems such as distributed communication and fault tolerance which I will touch on later.  

## A bit of history
{{< figure class="img-left" src="http://farm9.staticflickr.com/8432/7817529026_3168c8b8dc_d.jpg" alt="Swiss Bastard File" >}}
  
Early on in my career I worked solely in C#.  For years I toiled away using ASP.Net, WinForms, ADO.Net, etc before moving into back-end server-side problems.  I always found myself solving problems which other people shied away from, like multi-threading and performance profiling, so I spent another few years heavily engaged in server-side projects.  

Several years ago a colleague and I were brought in to fix an existing application which had to be scaled out to support thousands of concurrently connected clients.  It was written in C# to replace an aging C++ version which had also failed to deliver.  We found ourselves rapidly rewriting most of the core application from the inside out.  We developed a solid core framework before proceeding any further.  

I have always found that following design principles like [SOLID][solid] rather than blindly following patterns results in a well rounded solution.  The design that followed had a distinct functional nature, although we didn't know that at the time, in hindsight we had discovered functional programming for ourselves.

I cant remember exactly how it happened now, but at the time I was reading about Ocaml and F#.  This is when I really started to embrace functional programming, I started to tie together the design decisions we had made during the project against the functional programming concepts I was reading about.  
- - -
## Where am I now?
{{< figure src="http://farm8.staticflickr.com/7255/7817534648_bb65b0e5cc_c.jpg" >}}

Now that I have a better awareness of different styles I can more readily think through the problem at hand and tailor the solution in the right direction.   I'm now also using Clojure, Erlang, and Haskell.  Failure to at least be aware of different languages, what they are good at, and what problems they effectively solve is a big mistake in my eyes.  I try to do everything to the best of my ability, if your going to do something you might as well do it properly!  

Throughout my professional career various problems have came up at different times and by far the most time consuming problems are those in the realm of concurrency, scalability, and fault tolerance.  I have often found myself devoting large amounts of time to solving these issues, but there are languages which exist which deal with this at their core level - namely Erlang.  This is also something that I have been looking into recently.  

## This blog

This blog will be about using different tools for different jobs.  It will also be a place for me to have a rant about things which are bugging me.  Don't expect my usual pure F# output, expect posts on: Erlang, Clojure, Haskell, C#, or even, dare I say it, Javascript!  Don't worry though, there will still be plenty of F# content too.  

All the photos in this post are of my own tools taken by my wife Lynsey who is a keen photographer, you can find some more examples of her work on [Flickr][lynseysflickr].

Until next time...

[lynseysflickr]: http://www.flickr.com/photos/patchoulimemories
[logicprogramming]: http://en.wikipedia.org/wiki/Logical_programming
[functionalprogramming]: http://en.wikipedia.org/wiki/Functional_programming
[imperativeprogramming]: http://en.wikipedia.org/wiki/Imperative_programming
[objectorientedprogramming]: http://en.wikipedia.org/wiki/Object-oriented_programming
[solid]: http://en.wikipedia.org/wiki/SOLID_(object-oriented_design)

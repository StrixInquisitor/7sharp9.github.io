---
author: 7sharp9
title: Reckoning day
categories: [programming]
description: ""
type: post
comments: true
date: 2015-07-16
tags: [FSharp, Microsoft, MVP]
--- 
{{< figure src="https://farm1.staticflickr.com/256/18984716933_b7f2cbe6fb_t.jpg" title="" class="img-left" >}}
I've finally been awarded a [Microsoft MVP][1] its been quite a long time coming.  Ive been nominated for MVP every year since July 2011 but until now I have been unsuccessful.  On the 1st of July I received an email congratulating me that I had been successful on my MVP nomination.  
- - -
**I am now officially a Microsoft Most Valued Professional!**<!--more-->
{{< figure src="https://farm1.staticflickr.com/271/18984720063_55b4c8d2cf_z.jpg" >}}  

I was first nominated in July 2011 which seems like an age ago.  I have been 
nominated various times each year since then.  There are various activities that 
I have been involved in over the that last four or so years in F#.  Here are the 
ones that I feel particularly proud of:  
* * *
# Blogging

I write about various F# topics that I find interesting.  They are usually technical 
in nature or just what I find interesting or challenging at the time.  If I can 
help someone else figure something out from one of my posts, or bootstrap an 
interesting new project, then all the better!

# Renovating F# addin for MonoDevelop

When I first started using F# on OSX the MonoDevelop F# addin was pretty derelict, 
it hadn't been updated for sometime and didn't even compile.  It took quite a 
learning curve but I soon got up to speed and managed to improve things bit by bit.  

Editor support and general tooling can take a lot of time and effort, you can often 
get stuck on things with no clue on how to resolve something without digging deep 
within the F# compiler.  Im sure Don Syme has a spam filter on my emails by now :-)

# F# on IOS
Around January 2013 I managed to get F# working on iOS devices, you can read some 
of the details in [part one][2] and [part two][3].  It was really exciting to get 
a shinny new platform availably for F#!  

# Edge.fs
Before the FSharp.Compiler.Service even came to light I heard about [Edge.js][10] and 
wanted to have F# support, so in May 2013 I set about implementing it.  I 
ended up playing around in the compiler trying to figure how things worked.  You 
can read all about that [here][4].

# F# for ScriptCS
Around June 2013 [ScriptCS][9] was getting a great deal of publicity on Twitter so I 
decided it was about time F# should be [part of it][5].  This was also another 
opportunity to start hacking in the compiler again, ultimately this ended up with 
better programmatic REPL support added to the F# compiler.  

# Creation of F# Compiler Service

As part of working on the [F# plugin][8] for MonoDevelop it came to light that better 
support was needed for the F# compiler so that better tool integration could be 
built.  I worked to try and consolidate different areas of the the compiler that 
were being used by disparate tooling at the time.  This eventually lead to working 
with Don Syme to create [FSharp.Compiler.Service][7].  Over time this repository has 
been improved and lead to even better tooling for all editors and IDE's that use it.    

# FSharp.Core nuget package

I petitioned for some time to get better PCL support for F#, namely the quite 
common profiles 78 *(.NET Framework 4.5, Windows 8, Windows Phone Silverlight 8)* 
and 259 *(.NET Framework 4.5, Windows 8, Windows Phone 8.1, Windows Phone Silverlight 8)*.   Once 
the extra profiles were brought to fruition I created a nuget package to deploy 
them for easy consumption.   

# F# Community Bad Ass
This is one of my favorites:  In May 2014 I was awarded the F# community Badass 
award at [Functional Londoners][6]:
 
{{< tweet 464669331889340416 >}}

I wish I had got an official trophy for that, it would have looked really good on 
the mantlepiece!  :-)   



{{< figure src="https://farm1.staticflickr.com/256/18984716933_b7f2cbe6fb_z.jpg" title="" >}}  

Until next time ...

[1]: https://mvp.microsoft.com/en-us/default.aspx
[2]: http://7sharpnine.com/posts/monotouch-and-fsharp-part-i/
[3]: http://7sharpnine.com/posts/monotouch-and-fsharp-part-ii/
[4]: http://7sharpnine.com/posts/i-node-something/
[5]: http://7sharpnine.com/posts/can-i-have-some-fsharp-with-that/
[6]: http://www.meetup.com/FSharpLondon/
[7]: http://fsharp.github.io/FSharp.Compiler.Service/
[8]: https://github.com/fsharp/xamarin-monodevelop-fsharp-addin
[9]: http://scriptcs.net
[10]: http://tjanczuk.github.io/edge/#/

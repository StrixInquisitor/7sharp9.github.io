---
author: 7sharp9
layout: post
title: "F# 3.0 In The Mac And Mono World"
date: 2012-11-03
comments: true
categories: [Programming]
tags: [F#, tools, Mono, MonoDevelop, MonoGame]
---
So what have I been up to lately?  Well, lots of different things.  I have been taking it easy on the open source and blogging side of things as its been a hectic time of late in my personal life.  This seems to be changing now so I'm starting to get all of the ideas spinning around in my head into physical reality, or virtual reality, or what ever you want to call it.  Anyway, here's the first post on the subject of programming in F# using [MonoDevelop] and [Mono] natively on Macs.  <!-- more -->

{% img left http://www.mono-project.com/files/8/8d/Mono-gorilla-aqua.100px.png %}
I have always been a fan of [Mono] but I have always shied away from using it in anger, this is mainly due to my Windows heritage and using Visual Studio almost exclusively at work.  Times are changing though, and over the past few years I have been using Macs more and more.  In fact the only place where I still use Windows natively is at work, so it makes sense to get a development environment up and running to support me.  

Once the Beta of Mono 3.0 goes live it should contain within it a nice shiny new [F# 3.0][F#] installation, but until then we have to do a number of manual steps:

*   Install Mono 3.0
*   Install MonoDevelop
*   Compile and install F#3.0
*   Compile and install the MonoDevelop F# Language Binding

##Install Mono 3.0
Head over to the [Mono site] and install Mono 3.0 and the Mono MDK, they can be both found on the [download page][Mono].

IF you think you already have these installed you can check which version you have by Navigating to the directory: `/Library/Frameworks/Mono.framework/Versions`
You should see version 3.0 in the list:  

{% img https://lh3.googleusercontent.com/-N3cTmmXXvxI/UJUR1CIw4DI/AAAAAAAABgs/dQUFqgpQrvQ/s209/Mono3.0+install.png %}

Also ensure that the `Current` symbolic link points to Mono version 3.0, when I was first tried to get things running I couldn't understand why I kept getting a version mismatches, this was because my link was pointing to Mono 2.10.9.  

##Install MonoDevelop
I would recommend installing [MonoDevelop] straight from the packaged installer, as I write this the stable version is 3.0.4.7.  You can find the download page [here](http://monodevelop.com/Download).  

##Building F#3.0

These instruction can also be found [here][F#GitHub] but I thought it might be helpful to keep everything at hand here, if nothing else it will serve as a reference if I need to do this again or help anybody out.

{% codeblock lang:sh %}
#Make sure you have automake installed:
brew install automake

#First of all clone the F#3.0 repo from GitHub
git clone git://github.com/fsharp/fsharp.git\

#change the directory
cd fsharp

#run autogen script pointing at the current version of Mono
./autogen.sh --prefix=/Library/Frameworks/Mono.framework/Versions/Current/

#run make to compile the code
make

#run make to install
sudo make install
{% endcodeblock %}

Once that's all finished you can check everything is working by running `fsharpc`:

```
fsharpc
F# Compiler for F# 3.0 (Open Source Edition)
Freely distributed under the Apache 2.0 Open Source License

error FS0207: No inputs specified
```

If you didn't get a version 3.0 Compiler build displayed then check where `fsharpc` is running from by using the `type` command: 
```
type fsharpc
fsharpc is /usr/bin/fsharpc
```

You can now navigate there and check where the symbolic link is pointing to be using the `-l` parameter of the `ls` command:
{% codeblock lang:sh %}
cd /usr/bin
ls -l fsharpc
lrwxr-xr-x  1 root  wheel  51  3 Nov 09:33 fsharpc -> /Library/Frameworks/Mono.framework/Commands/fsharpc
{% endcodeblock %}

When I first tried to get the F# 3.0 compiler installed I had all sorts of problem with the symbolic link pointing to the old version.  I don't know whether the installer had failed or whether an old version of the F# compiler or the F# language binding had caused issues, the process was so much smoother on my MacBook Pro as it was a clean install.

##Build the MonoDevelop F# language binding

{% codeblock lang:sh %}
git clone git://github.com/fsharp/fsharpbinding.git
cd fsharpbinding
./configure.sh
make
make install
{% endcodeblock %}

That's it we're all done!  Pretty easy stuff thanks to [Ben Winkel] and others for putting in some time to fix up the issues.  

Now you should be able to fire up [MonoDevelop] and start building some F# code!

You can check the add-in is installed properly my opening the Add-in Manager from the [MonoDevelop] menu.  The F# language binding should be shown as:

{% img https://lh5.googleusercontent.com/-ubaW6iROJyc/UJUR1MKOaJI/AAAAAAAABgw/PgUNUmFHhBM/s791/Addin+manager.png %}

When you start a new solution you should now be presented with some F# specific options:

{% img https://lh4.googleusercontent.com/-p3GdkcD14aA/UJUR1WHLuhI/AAAAAAAABg0/TnpTuEkuC30/s932/New+Solution.png %}

Next time I will take this a step further by showing how to build, install and integrate [MonoGame] with F#.  I will also be releasing an F# specific project template for use with [MonoDevelop].

Until next time...

[Ben Winkel]: http://twitter.com/winkler_ben
[F#]: http://msdn.microsoft.com/en-us/vstudio/hh388569
[F#GitHub]: https://github.com/fsharp/fsharp
[Mono]: http://www.mono-project.com/What_is_Mono
[MonoDevelop]: http://monodevelop.com
[Mono Site]: http://www.go-mono.com/mono-downloads/download.html
[MonoGame]: http://monogame.codeplex.com
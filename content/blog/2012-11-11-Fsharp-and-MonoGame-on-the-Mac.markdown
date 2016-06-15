---
author: 7sharp9
layout: post
title: "F# and MonoGame on the Mac"
date: 2012-11-11
comments: true
categories: [programming]
tags: [F#, tools, Mono, MonoDevelop, MonoGame, Mac]
description: ""
type: post
---
What we are going to do in this post is take a whistle stop tour of getting [MonoGame][0] up and running along with a 
simple demo in F#.  Over the last few days I have been building an F# project template for MonoDevelop, this post will 
also how to get that installed too.  <!-- more -->

First of all I'm going to assume that you have the following installed:

*   Mono 3.0 beta
*   MonoDevelop 
*   MonoDevelop F# language binding

If you don't have a look at my [previous post][1] that explains all that, if you don't want to build the F# binding from source 
then you can use the `Add-in Manager`.  If you look in `Gallery` the language binding section contains the F# language 
binding.  I prefer using the source at the moment as I like to tweak a few things here and there and nosy around in the code.  

## Cloning and building
First clone my [MonoGame repo][2], why my repo?  Well, I have been doing F# specific work and I have been submitting my 
pull requests but there will always be a lag while I'm waiting for one of the maintainers to merge in my code, an I might 
fix a bug in between writing this blog and you reading it.  The main repo is located [here][3] if you are interested in 
looking at that too.  

```
git clone git@github.com:7sharp9/MonoGame.git
```

OK, now we need to initialise and get the submodules:

```
git submodule init
git subbmodule update
```

Now go and make a cup of tea or coffee or something because this will take a little while! :-)

if you get any serious errors from the submodules you cold try this:

```
git submodule sync
git submodule update
```

Check that the main framework builds, this will test if all the submodules updated correctly as well.  

```
cd MonoGame
xbuild MonoGame.Framework.MacOS.sln
```

## Building the Project templates

I was going to go through the manual steps required to get things up and running but I figured I may as well go through 
getting the project templates installed too.  Now the project templates are not 100% by any means and there is currently 
work under way to produce a cross platform installer that will take care of everything.  

There are currently the following C# based templates:

*   MonoGame for Android
*   MonoGame for iOS
*   MonoGame for Windows Application
*   MonoGame for Linux
*   MonoGame for Mac Application

And the following brand new shiny F# templates that I added a few days ago:

*   MonoGame for Mac Application

Yeah I know there's only one for now but Ill get there...  

Right, we're going to rattle through some commands to build the project templates:

```
#Change directory to the project templates
cd ProjectTemplates/MonoDevelop/MonoDevelop.MonoGame/

#Build the Project templates solution:

xbuild MonoDevelop.MonoGame.sln 
```

Now we're going to create the add-in structure and install it in MonoDevelop

```
#create a folder for the add-in:
mkdir /Applications/MonoDevelop.app/Contents/MacOS/lib/monodevelop/AddIns/MonoDevelop.MonoGame

#Copy the templates and icons into that folder:
cp -R MonoDevelop.MonoGame/templates /Applications/MonoDevelop.app/Contents/MacOS/lib/monodevelop/AddIns/MonoDevelop.MonoGame
cp -R MonoDevelop.MonoGame/icons /Applications/MonoDevelop.app/Contents/MacOS/lib/monodevelop/AddIns/MonoDevelop.MonoGame
cp -R MonoDevelop.MonoGame/bin/Release/MonoDevelop.MonoGame.dll /Applications/MonoDevelop.app/Contents/MacOS/lib/monodevelop/AddIns/MonoDevelop.MonoGame
```

If you fire up MonoDevelop and create a new solution now you should see a new project type:

{{< figure src="https://lh5.googleusercontent.com/-5_khSpX_ong/UJ-MeJOyTpI/AAAAAAAABh4/oftw3zWSMqo/s1062/MonoGame+Mac+Application.png" >}}

I know this is a bit long winded but it will get you started until a proper installer finished.

Oh yeah there are currently a couple of caveats, once you have created a new project from the template you **must** do 
the following:  

>   Open the project references and re-add the **Lidgren.Network.dll** and the **MonoGame.FrameWork.MacOS.dll** references.  

I know I know that's a **major pain** but let me tell you the reason for this.  

### Package references in MonoDevelop

You might of noticed that the aforementioned references were shown in red when the new project was opened, this was 
because there was no way for MonoDevelop to know where they were currently located.  We could of added a 
[package config file][5] to the `/Library/Frameworks/Mono.framework/Versions/3.0.0/lib/pkgconfig` folder which would 
allow MonoDevelop to know how to resolve the references but this then leads to a further problem:

Package references in MonoDevelop are expected to be located in the GAC, and because of this assumption the copy local property has no 
effect.  There is an [oustanding bug][4] logged for this issue.  There are workarounds for these issues as mentioned in 
the [bug report][4] but it is still currently an issue in MonoGame.  Hopefully it will be addressed in the near future.  

You might be asking why cant we put the assemblies it in the GAC?  

Well we could do although you would have then have to sign the assembly as it is currently not strongly named and then 
install it in the GAC with `gacutil`, and as this is still an early beta so I'm happy to keep the assembly out of the GAC for now.  

- - - 
You should be ready to put MonoGame to use now.  If you run the template project you will get a MonoGame logo drawn to 
the screen.  I suggest looking at some of the demo's in the Samples folder and maybe try porting a couple of them over 
to F#.  In future posts Ill try to address some of the functional aspect of using F# with MonoGame.  

## What's next?
I'm currently starting work on a post which shows F# using some of the 3D aspects of MonoGame.  Exciting!

Until next time...

[0]: http://monogame.codeplex.com
[1]: http://7sharpnine.com/posts/Fsharp-3-in-the-Mac-and-Mono-World/
[2]: https://github.com/7sharp9/MonoGame
[3]: https://github.com/mono/MonoGame
[4]: https://bugzilla.xamarin.com/show_bug.cgi?id=4030
[5]: http://en.wikipedia.org/wiki/Pkg-config

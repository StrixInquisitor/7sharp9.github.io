---
author: 7sharp9
layout: post
title: "MonoTouch and F# part II"
date: 2013-02-07
comments: true
categories: [programming]
tags: [FSharp, tools, Mono, MonoDevelop, MonoTouch, Xamarin, iOS]
description: ""
type: post
---
In the last post we left at the point where everything was running fine and dandy on the Simulator.  So what happens if we compile for the real hardware?  

Lets change the active configuration to `Debug|iPhone` and hit build, what do we get?<!-- more -->

### Boom!  

*Error MT2002: Could not resolve: FSharp.Core, Version=4.3.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a (MT2002) (singleview)*

So I guess we need to tell it where the FSharp.Core.dll is, lets add a reference to it:

	/Libraries/FrameWorks/Mono.FrameWork/Libraries/mono/Microsoft F#/v4.0/Fsharp.Core.dll

Now try and build ... another weird error:

*unknown-file(1,1): Error FS2020: The assembly 'F#/v4.0/FSharp.Core.dll' is listed on the command line. Assemblies should be referenced using a command line flag such as '-r'. (FS2020) (singleview)*

Hmmm, lets look in the F# compiler source, were going to have to break out the big guns for this one.  

What we need to do is look at the different targets that are available.  I remember seeing different targets when I was nosing through the source files a while ago.  Also if you look at the `readme.md` file that comes with the open source compiler:

You can also build FSharp.Core for: .NET 2.0, Mono 2.1, Silverlight 5.0, Portable Profile47 (net4+sl4+wp71+win8) and XNA 4.0 for Xbox 360 profiles:

```
msbuild fsharp-library-build.proj /p:TargetFramework=net20 
msbuild fsharp-library-build.proj /p:TargetFramework=mono21
msbuild fsharp-library-build.proj /p:TargetFramework=portable-net4+sl4+wp71+win8
msbuild fsharp-library-build.proj /p:TargetFramework=sl5
msbuild fsharp-library-build.proj /p:TargetFramework=net40-xna40-xbox360
```

So lets build the `mono21` target with `xbuild`: 

```
xbuild fsharp-library-build.proj /p:TargetFramework=mono21
```
Now that's build lets reference the output and see what happens:  

Arrgh another error this time relating the the version of the framework that we have compiled against.  

If you read the documentation for [MonoTouch][1] in a little more detail you will discover that a different mscorlib is required.  We need to modify this in the build script:

Open up `FSharp.Source.Targets` and find the `<PropertyGroup Condition="'$(TargetFramework)'=='mono21'">` section, add the following after the `<DefineConstants>` elements.

	<OtherFlags>$(OtherFlags) --simpleresolution -r:"/Developer/MonoTouch/usr/lib/mono/2.1/mscorlib-runtime.dll"  </OtherFlags>

Right, fingers crossed...

Sigh, another error:

*Error MT2002: Can not resolve reference: System.Reflection.Emit.AssemblyBuilder (MT2002) (singleview)*

Were getting closer though.

Lets look at the `<DefineConstants/>` that are declared in the build file, if you have a quick look you will notice that there is one called `FX_NO_REFLECTION_EMIT` that's what we need so that `Reflection.Emit` is not included.  MonoTouch does not support `Reflection.Emit` due to the meta data not being available once the code have been compiled with the [AOT compiler][3].  

Lets add that constant to the end of the rest:  

	<DefineConstants>$(DefineConstants);FX_NO_REFLECTION_EMIT</DefineConstants>

If we rebuild `Fsharp.Core` again with `xbuild` and rebind the reference in our test project...

### Wow it works!

You should now have a working hello world application that can be deployed and run on real hardware.  

## Final Words

As this is just a documented hackathon I have mainly brain dumped what I remembered doing after the fact, so some steps may be slightly different.  As soon as time permits Ill be adding a couple of project templates to the [FSharpBinding][2] to allow building F# MonoTouch libraries and applications.  

I also have some ideas for dealing with the UI and tooling with [Xcode][4] but Ill need a little time to investigate to see if it's a viable option...

{{< EssentialListening
    "|http://upload.wikimedia.org/wikipedia/en/4/44/Soilscars.jpg|Soil - Scars"
    "|http://upload.wikimedia.org/wikipedia/en/e/e9/AlbinoSlug.jpg|Buckethead - Albino Slug" >}}  
Until next time!

[1]: http://xamarin.com/monotouch
[2]: https://github.com/fsharp/fsharpbinding
[3]: http://www.mono-project.com/AOT
[4]: https://developer.apple.com/technologies/tools/

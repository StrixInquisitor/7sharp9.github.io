---
author: 7sharp9
layout: post
title: "MonoTouch and F# part I"
date: 2013-02-03
comments: true
categories: [programming]
tags: [FSharp, tools, Mono, MonoDevelop, MonoTouch, Xamarin, iOS]
description: ""
type: post
---
[MonoTouch][2] and F# that would be a cool duo right?  

Well let me explain what needs to be done and why to get this pair working together.  

I heard rumours a while ago that F# and [MonoTouch][2] would not play together nicely because of [limitations][1] in the ahead of time compilation [(AOT)][5].  So I thought I would either prove or disprove this with some concentrated hacking.  How hard can it be?   

As my good friend and colleague Dr. Kewin would quote:

> “No problem can withstand the assault of sustained thinking.”—Voltaire<!-- more -->

## Prerequisites 
These are the same as MonoTouch, I'm using a Mac and MonoDevelop at the moment.  You would need a Mac anyway to be able to do the compile and deploy to an iOS device.  [Xcode][3] with an Apple profile and certificates are required for code signing etc.  
## First steps
So how do we tackle this?   

First lets look at the **C# Single View** MonoTouch project file (`.csproj`) up to the end of the first PropertyGroup:

```
<?xml version="1.0" encoding="utf-8"?>
<Project DefaultTargets="Build" ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <Configuration Condition=" '$(Configuration)' == '' ">Debug</Configuration>
    <Platform Condition=" '$(Platform)' == '' ">iPhoneSimulator</Platform>
    <ProductVersion>10.0.0</ProductVersion>
    <SchemaVersion>2.0</SchemaVersion>
    <ProjectGuid>{822346B5-6805-42FD-9B6A-65446A688E63}</ProjectGuid>
    <ProjectTypeGuids>{6BC8ED88-2882-458C-8E55-DFD12B67127B};
                      {FAE04EC0-301F-11D3-BF4B-00C04F79EFBC}</ProjectTypeGuids>
    <OutputType>Exe</OutputType>
    <RootNamespace>HelloWorld</RootNamespace>
    <AssemblyName>HelloWorld</AssemblyName>
  </PropertyGroup>
```

 The bits we are interested in are the **ProjectTypeGuids**.  Visual Studio/MonoDevelop projects use these guid's to identify the type of the project.  If you do a bit of Googling *(or Binging...)* you would find that:

*   **6BC8ED88-2882-458C-8E55-DFD12B67127B**  is a MonoTouch project type guid
*   **FAE04EC0-301F-11D3-BF4B-00C04F79EFBC**  is a C# project type guid

The F# project type guid is **F2A71F9B-5D33-465A-A702-920D77279786**.  We can now replace **FAE04EC0-301F-11D3-BF4B-00C04F79EFBC** with the F# one.  For a comprehensive list of project type guid's have a look at [Mikhail Pilin's blog][4].  Next scroll down to the bottom of the project file and update the  <Import Project="$(MSBuildBinPath)\Microsoft.CSharp.targets" /> to <Import Project="$(MSBuildExtensionsPath32)\..\Microsoft F#\v4.0\Microsoft.FSharp.Targets" />.  the final step on the project file is to change the project file extension from `.csproj` to `.fsproj`.   

The last of the tweaking is to open up the `.sln` file and make a slight change to that too:

	Microsoft Visual Studio Solution File, Format Version 11.00
	# Visual Studio 2010
	Project("{FAE04EC0-301F-11D3-BF4B-00C04F79EFBC}") = "singleview", "singleview\singleview.fsproj", "{4465399C-4EE8-4F60-AD9A-EB9AEDD1C5BF}"
	EndProject
	Global
	...snip...

Modify the  Project sections Guid **FAE04EC0-301F-11D3-BF4B-00C04F79EFBC** to the F# project type Guid **4925A630-B079-445d-BCD4-3A9C94FE9307**.  If you forget this step then MonoDevelop will get really confused and try to compile the F# project with the C# compiler.  

## Code Changes
For the sake of simplicity I'm going to port the C# code verbatim showing the C# code first then the F# code.  The easiest way would probably be to change all the  C# files to have the `.fs` extension and edit them in place, remembering to also update the entries in the `.fsproj` file too this only takes a second to do.  

I know what you are going to say: "Why didn't you just create a nice project template for us all to use?"

I am, I am, patience!  

A number of people wanted to know what I did to get things going so this is my documented 'hack-a-thon' if you like.  The project template will be along shortly.  Lets move along to the code changes.  

## ViewController

```
using System;
using System.Drawing;
using MonoTouch.Foundation;
using MonoTouch.UIKit;

namespace singleview
{
	public partial class singleviewViewController : UIViewController
	{
		public singleviewViewController () : base ("singleviewViewController", null)
		{
		}
		
		public override void DidReceiveMemoryWarning ()
		{
			// Releases the view if it doesn't have a superview.
			base.DidReceiveMemoryWarning ();
			// Release any cached data, images, etc that aren't in use.
		}
		
		public override void ViewDidLoad ()
		{
			base.ViewDidLoad ();
			// Perform any additional setup after loading the view, typically from a nib.
		}
		
		public override void ViewDidUnload ()
		{
			base.ViewDidUnload ();
			// Clear any references to subviews of the main view in order to
			// allow the Garbage Collector to collect them sooner.
			// e.g. myOutlet.Dispose (); myOutlet = null;
			ReleaseDesignerOutlets ();
		}
		
		public override bool ShouldAutorotateToInterfaceOrientation (UIInterfaceOrientation toInterfaceOrientation)
		{
			// Return true for supported orientations
			return (toInterfaceOrientation != UIInterfaceOrientation.PortraitUpsideDown);
		}
	}
}

// This file has been generated automatically by MonoDevelop to store outlets and
// actions made in the Xcode designer. If it is removed, they will be lost.
// Manual changes to this file may not be handled correctly.
using MonoTouch.Foundation;

namespace singleview
{
	[Register ("singleviewViewController")]
	partial class singleviewViewController
	{
		void ReleaseDesignerOutlets ()
		{
		}
	}
}
```

```fsharp
namespace Singleview

open System
open System.Drawing
open MonoTouch.Foundation
open MonoTouch.UIKit

[<Register ("singleviewViewController")>]
type singleviewViewController() =
    inherit UIViewController("singleviewViewController", null)
        
    let ReleaseDesignerOutlets() = ( (* No outlets to release  *))

    override x.DidReceiveMemoryWarning() =
    // Releases the view if it doesn't have a superview.
        base.DidReceiveMemoryWarning();
        // Release any cached data, images, etc that aren't in use.

    override x.ViewDidLoad() =
        base.ViewDidLoad()
        // Perform any additional setup after loading the view, typically from a nib.

    override x.ViewDidUnload() =
        base.ViewDidUnload()
        // Clear any references to subviews of the main view in order to
        // allow the Garbage Collector to collect them sooner.
        // e.g. myOutlet.Dispose (); myOutlet = null;
        ReleaseDesignerOutlets()

    override x.ShouldAutorotateToInterfaceOrientation(toInterfaceOrientation) =
        // Return true for supported orientations
        toInterfaceOrientation <> UIInterfaceOrientation.PortraitUpsideDown
```

On looking at this section you will notice that there is no partial class in the F# version, that's because F# doesn't have any notion of partial classes.  In this simple project we don't actually have any interaction with the UI so designer interaction is a moot point at the moment.   

The `.fsproj` file still needs to be edited to remove the nested partial class that is present in the C# version:

```
   <Compile Include="singleviewViewController.designer.cs">
      <DependentUpon>singleviewViewController.cs</DependentUpon>
    </Compile>
```

Simply remove the `DependUpon` element and just use the name `singleviewViewController.fs`:

```
<Compile>singleviewViewController.fs/>
```

The lack of partial classes in F# makes the tooling available for UI designer a pain to integrate tightly into F# without a bit of work work *(I have some ideas on that that I'm currently experimenting with that Ill return to after finishing this article)*.  Currently MonoTouch uses the Xcodes interface designer to build the UI which is stored in a xib file.  This is simply a file describing the user interface and its interaction points.  The Properties of the UI are called `Outlets` and events spawned from the UI are called `Actions`.  

## AppDelegate

```
using System;
using System.Collections.Generic;
using System.Linq;
using MonoTouch.Foundation;
using MonoTouch.UIKit;

namespace singleview
{
	// The UIApplicationDelegate for the application. This class is responsible for launching the 
	// User Interface of the application, as well as listening (and optionally responding) to 
	// application events from iOS.
	[Register ("AppDelegate")]
	public partial class AppDelegate : UIApplicationDelegate
	{
		// class-level declarations
		UIWindow window;
		singleviewViewController viewController;

		// This method is invoked when the application has loaded and is ready to run. In this 
		// method you should instantiate the window, load the UI into it and then make the window visible.
		// You have 17 seconds to return from this method, or iOS will terminate your application.
		public override bool FinishedLaunching (UIApplication app, NSDictionary options)
		{
			window = new UIWindow (UIScreen.MainScreen.Bounds);
			
			viewController = new singleviewViewController ();
			window.RootViewController = viewController;
			window.MakeKeyAndVisible ();
			
			return true;
		}
	}
}
```

```fsharp
namespace Singleview
open System
open System.Collections.Generic
open MonoTouch.Foundation
open MonoTouch.UIKit

// The UIApplicationDelegate for the application. This class is responsible for launching the 
// User Interface of the application, as well as listening (and optionally responding) to application events from iOS.
[<Register ("AppDelegate")>]
type AppDelegate() =
    inherit UIApplicationDelegate()
    
    let mutable window = Unchecked.defaultof<_>
    let mutable viewController = Unchecked.defaultof<_>

    // This method is invoked when the application has loaded and is ready to run. In this 
    // method you should instantiate the window, load the UI into it and then make the window visible.
    // You have 17 seconds to return from this method, or iOS will terminate your application.
    override x.FinishedLaunching ( app: UIApplication,  options: NSDictionary) =
        window <- new UIWindow(UIScreen.MainScreen.Bounds)
        viewController <- new singleviewViewController()
        window.RootViewController <- viewController
        window.MakeKeyAndVisible()
        true
```

The code is pretty similar between the two implementations, with the F# version omitting the type annotations, semicolons and curly braces.  The other area to notice is that the `mutable` variable declarations for the `window` and `viewController` bindings.  The C# implementation defaults to mutable variables whereas F# defaults to the safer immutable ones.   

## Program/main

```
using System;
using System.Collections.Generic;
using System.Linq;
using MonoTouch.Foundation;
using MonoTouch.UIKit;

namespace singleview
{
	public class Application
	{
		// This is the main entry point of the application.
		static void Main (string[] args)
		{
			// if you want to use a different Application Delegate class from "AppDelegate"
			// you can specify it here.
			UIApplication.Main (args, null, "AppDelegate");
		}
	}
}
```

```fsharp
module main
open System
open System.Collections.Generic
open MonoTouch.Foundation
open MonoTouch.UIKit

    [<EntryPoint>]
    let main( args) = 
        UIApplication.Main (args, null, "AppDelegate")
        0
```

The main thing you will notice is that the F# code is terser, again dropping the type annotations, semicolons and curly braces.  Oh, I also called the entry point main.  To be precise it's a function called main in a module named main, there's no need to create a class or type for this.   

## The Xib file
In C# MonoToch projects the `xib` file is compiled and embedded for you as part of the build process, unfortunately this is not currently possible in F# so we have to do it manually.  In an ideal world this would all be done by the F# project at build time and this is something that I'm working on too.  In the mean time we have to do it manually so open up your trusty friend the `Terminal`.  

I'm going to split the command line into separate parts due to its size:

First of all we invoke the `ibtool`:  

	/Applications/Xcode.app/Contents/Developer/usr/bin/ibtool --errors --warnings --notices --output-format human-readable-text --compile

Followed by name of the `.nib` file you want to compile to:

	"/yourPath/singleviewViewController.nib"

The path of the `.xib` you want to compile from:

	"/yourPath/singleviewViewController.xib"

Finally the sdk that you want to use for compilation, in this instance it is The iPhoneSimulator6.0.sdk as we are targetting the simulator: 
	--sdk "/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator6.0.sdk"

Once you have compiled to a `.nib` file include it in the project, set the build action to `Content`.  You can still include the `.xib` version within the project if you want but you would have to set the build action to `None`.  Currently the F# binding does not support the build action of `Interface Definition` if it did then we probably wouldn't have to go through the manual compilation process either.  

That ought to do it, everything should now work on the simulator.  If you try to compile to a real phone then everything will quickly come grinding to a halt but Ill explain all of that next time and how to resolve it too.  

Until next time!

{{< EssentialListening
    "|http://upload.wikimedia.org/wikipedia/en/5/5f/MegadethThirteen.jpg|Megadeth - TH1RT3EN"
    "|http://upload.wikimedia.org/wikipedia/en/7/7a/Worship_Music.jpg|Anthrax - Worship Music" >}}  

[1]: http://docs.xamarin.com/ios/about/limitations
[2]: http://xamarin.com/monotouch
[3]: https://developer.apple.com/technologies/tools/
[4]: http://workblog.pilin.name/2012/11/visual-studio-project-type-guids.html
[5]: http://www.mono-project.com/AOT
[7]: http://xamarin.com

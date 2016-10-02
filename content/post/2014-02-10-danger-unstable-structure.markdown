---
author: 7sharp9
layout: post
title: "Danger unstable structure - No more!"
date: 2014-02-10
categories: [programming]
tags: [FSharp, Xamarin, tools]
comments: true
description: ""
type: post
---
{{< figure class="img-left sixth" src="https://lh4.googleusercontent.com/-dJcVwxtDxw8/Uvaxn7dezqI/AAAAAAAAByo/QffHElmM96I/s512-no/IMG_0379.png" >}}
  
Over the last year a lot of work has been done on the [F# addin][4] for [Xamarin Studio][5].  Lots of great new features have been added and a lot of bugs have been squashed.  I want to talk a bit about whats been happening and the evolution of the F# addin.  
  
<!-- more -->  

The F# addin for [MonoDevelop][2] / [Xamarin Studio][5] hasn't always been as stable and pretty as it is now.  When I first started working on it I went through a rocky phase of just trying to get it to compile and install.  There were a whole host of changes going on in MonoDevelop which caused a lot of head scratching and late nights.  Of all the issues encountered tooltips were the absolute bane of my life, they were often horribly mangled like this:   

{{< figure src="https://lh5.googleusercontent.com/-12p4uUpRK4g/Uvjtt2WAe7I/AAAAAAAABzs/QaQ-Tw8jIDw/w924-h416-no/77e93b30-7fa2-11e2-9490-303b1f7cb8b3.png" >}}

Or completion lists were insanely long and downright pugly:  
{{< figure src="https://lh5.googleusercontent.com/-yZCVc4ymsGg/Uvsx_YwKbkI/AAAAAAAAB0w/NSvcqUyoC7w/w1400-h958-no/huge_overloads.png" >}}

Modern IDE's have features like tooltips and auto completions which are so intrinsic that without them you feel a little lost without them, instead you have to rely on having eidetic memory of the various APIs and functions.  Over the course of fixing bugs and adding new features inevitably you end up breaking lots of things which can make for a frustrating experience with half mangled IDE.  Thankfully all these issues have now been addressed and lots of new features have also been added.  Here are all the new features and improvements that have gone in over the last year or so.  

### UI Improvements

#### Rename refactoring
One of the most recent and exciting new features is rename refactoring.  Renaming can be achieved by either using a shortcut key `Meta R` or by right clicking and selecting rename from the context menu.  Here's a screen cast of rename refactoring in action:  
<object id="scPlayer"  width="700" height="530" type="application/x-shockwave-flash" data="http://content.screencast.com/users/DaveThomas2126/folders/Default/media/7d3ff591-57fe-4516-b888-5ee5c1487524/scplayer.swf" >
<param name="movie" value="http://content.screencast.com/users/DaveThomas2126/folders/Default/media/7d3ff591-57fe-4516-b888-5ee5c1487524/scplayer.swf" />
<param name="quality" value="medium" />
<param name="bgcolor" value="#FFFFFF" />
<param name="flashVars" value="thumb=http://content.screencast.com/users/DaveThomas2126/folders/Default/media/7d3ff591-57fe-4516-b888-5ee5c1487524/FirstFrame.jpg&containerwidth=780&containerheight=610&xmp=sc.xmp&content=http://content.screencast.com/users/DaveThomas2126/folders/Default/media/7d3ff591-57fe-4516-b888-5ee5c1487524/ScreenFlow.mp4&blurover=false" />
<param name="allowFullScreen" value="false" />
<param name="scale" value="showall" />
<param name="allowScriptAccess" value="always" />
<param name="base" value="http://content.screencast.com/users/DaveThomas2126/folders/Default/media/7d3ff591-57fe-4516-b888-5ee5c1487524/" />
<iframe type="text/html" frameborder="0" scrolling="no" style="overflow:hidden;" src="http://www.screencast.com/users/DaveThomas2126/folders/Default/media/7d3ff591-57fe-4516-b888-5ee5c1487524/embed" ></iframe>
</object> 

#### Highlight usages
The precursor to rename refactoring was highlight usages, by clicking in the code editor the current identifier is highlighted and all occurrences of of that identifier are also highlighted.  This makes it very easy to see where a specific identifier is used and is one of my favorite features:  

{{< figure src="https://lh5.googleusercontent.com/-Adx7bMfd7O0/UvtEs9UQdcI/AAAAAAAAB1w/aMva_sHDX1U/w880-h498-no/Screen+Shot+2014-02-12+at+09.51.17.png" >}}

#### Tooltip overload refinements
Overload tooltips have now been refined so they look like the image below rather than the huge overload list shown at the top of this post.  
{{< figure src="https://lh3.googleusercontent.com/-hnmNtHIsrRU/UvtOB_yf0qI/AAAAAAAAB2U/FY3cMm-SY-0/w1400-h324-no/overload_enhancement.png" >}}

#### Tooltip arrow indicators
A little arrow indicators is now shown and positioned over the center of the tooltip.  Yeah is only cosmetic but it was annoying having it missing.  

#### Symbol tooltips
We now also have tooltips for symbols, which makes custom symbol use more palatable.  Now you can simply hover over the symbol see see its signature and xmldoc summary.  
{{< figure src="https://lh4.googleusercontent.com/-FFZtSnVdSkU/UvtVTNb5KlI/AAAAAAAAB2w/uPCpJhuOaoY/w642-h204-no/symbol+tooltips.png" >}}

#### Drag and drop file ordering
F# source files can now be dragged in the Solution pad for correct ordering.  This is limited to ordering files within a single folder at the moment. Source files can still be moved from one folder to another but they must be ordered as a separate operation.   

#### Pathed document navigation
Pathed document navigation is implemented in the source window to aid navigation between modules, types and functions within a source file:  

{{< figure src="https://lh3.googleusercontent.com/-Evxt2R9Whr8/Uvt81UNJImI/AAAAAAAAB3M/_de-clz_b80/w1400-h878-no/pathed_doc.png" >}}

### Interactive Improvements

#### Send all project references to FSI
From the context menu or keyboard shortcut `Ctrl Meta P` you can choose to send all of the current projects reference to F# interactive, this saves having to enter a bunch of #r statements.  

#### Send line moves down automatically
For doing demos and testing scripts you can use a keyboard shortcut to sent the current line to F# interactive, the caret automatically drops down to the next line which makes stepping through the script a pinch.  

#### Clear/Reset FSI keyboard shortcuts
By pressing `Ctrl Meta R` you can reset the current instance of F# interactive, similarly `Ctrl Meta C` will clear the current F# interactive window.  

#### FSI theming
You can now also choose to have F# interactive match you current theme colours or pick your own:  
{{< figure class="6u" src="https://lh4.googleusercontent.com/-81Lw7ySfNRE/Uvs6_xH8pcI/AAAAAAAAB1Q/eO1Y2LmbT_E/w1400-h874-no/themed_fsi.png" >}}  

### Engineering improvements

Originally we had a reflective binding to the F# compiler services which meant it was version agnostic but quite difficult to debug and add new features to.  

Actually just for posterity, and you might find this useful/interesting heres the code, it uses the [dynamic lookup operator][9] `(?)` :

```fsharp
let (?) (o:obj) name : 'R =
  // The return type is a function, which means that we want to invoke a method
  if FSharpType.IsFunction(typeof<'R>) then
    let argType, resType = FSharpType.GetFunctionElements(typeof<'R>)
    FSharpValue.MakeFunction(typeof<'R>, fun args ->
      // We treat elements of a tuple passed as argument as a list of arguments
      // When the 'o' object is 'System.Type', we call static methods
      let methods, instance, args, owner = 
        let args = 
          if Object.Equals(argType, typeof<unit>) then [| |]
          elif not(FSharpType.IsTuple(argType)) then [| args |]
          else FSharpValue.GetTupleFields(args)
        if (typeof<System.Type>).IsAssignableFrom(o.GetType()) then 
          let methods = (unbox<Type> o).GetMethods(staticFlags) |> Array.map asMethodBase
          let ctors = (unbox<Type> o).GetConstructors(ctorFlags) |> Array.map asMethodBase
          let owner = (unbox<Type> o).Name + " (static)"
          Array.concat [ methods; ctors ], null, args, owner
        else 
          let owner = o.GetType().Name + " (instance)"
          o.GetType().GetMethods(instanceFlags) |> Array.map asMethodBase, o, args, owner
      
      // A simple overload resolution based on the name and number of parameters only
      let methods = 
        [ for m in methods do
            if m.Name = name && m.GetParameters().Length = args.Length then yield m 
            if m.Name = name && m.IsGenericMethod &&
               m.GetGenericArguments().Length + m.GetParameters().Length = args.Length then yield m ]
      match methods with 
      | [] -> failwithf "No method '%s' with %d arguments found in %s" name args.Length owner
      | _::_::_ -> failwithf "Multiple methods '%s' with %d arguments found %s" name args.Length owner
      | [:? ConstructorInfo as c] -> c.Invoke(args)
      | [ m ] when m.IsGenericMethod ->
          let tyCount = m.GetGenericArguments().Length
          let tyArgs = args |> Seq.take tyCount 
          let actualArgs = args |> Seq.skip tyCount
          let gm = (m :?> MethodInfo).MakeGenericMethod [| for a in tyArgs -> unbox a |]
          gm.Invoke(instance, Array.ofSeq actualArgs)
      | [ m ] -> m.Invoke(instance, args) ) |> unbox<'R>
  else
    // When the 'o' object is 'System.Type', we access static properties
    let typ, flags, instance = 
      if (typeof<System.Type>).IsAssignableFrom(o.GetType()) then unbox o, staticFlags, null
      else o.GetType(), instanceFlags, o
    
    // Find a property that we can call and get the value
    let prop = typ.GetProperty(name, flags)
    if Object.Equals(prop, null) then 
      // Find a field that we can read
      let fld = typ.GetField(name, flags)
      if Object.Equals(fld, null) then
        // Try nested type...
        let nested = typ.Assembly.GetType(typ.FullName + "+" + name)
        if Object.Equals(nested, null) then 
          failwithf "Property, field or nested type '%s' not found in '%s' using flags '%A'." name typ.Name flags
        elif not ((typeof<'R>).IsAssignableFrom(typeof<System.Type>)) then
          failwithf "Cannot return nested type '%s' as value of type '%s'." nested.Name (typeof<'R>.Name)
        else nested |> box |> unbox<'R>
      else
        // Get field value
        fld.GetValue(instance) |> unbox<'R>
    else
      // Call property
      let meth = prop.GetGetMethod(true)
      if prop = null then failwithf "Property '%s' found, but doesn't have 'get' method." name
      try meth.Invoke(instance, [| |]) |> unbox<'R>
      with err -> failwithf "Failed to get value of '%s' property (of type '%s'), error: %s" name typ.Name (err.ToString())
```

To use it a type wrapper had to be constructed a bit like this:
```fsharp
let interactiveCheckerType = asmCompiler.GetType("Microsoft.FSharp.Compiler.SourceCodeServices.InteractiveChecker")

type InteractiveChecker(wrapped:obj) =
member x.TryGetRecentTypeCheckResultsForFile(filename:string, options:CheckOptions) =
   let res = wrapped?TryGetRecentTypeCheckResultsForFile(filename, options.Wrapped) : obj
   if res = null then None else
      let tuple = res?Value
      Some(UntypedParseInfo(tuple?Item1), TypeCheckResults(tuple?Item2), int tuple?Item3)
```

It was quite fiddly to get right as `FSharp.Core` had to be used reflectively as you would potentially be using a different version of `FSharp.Core` depending on what version of F# you were binding to.  I think this kind of reflective calling could be simplified by creating an F# type provider, I can think of a variety of uses for something like this.   

Ultimately a branch called **hardbinding** was created which led to the creation of the F# Compiler Editor branch of the F# compiler.  Slowly over the last few months that evolved into the [FSharp Compiler Service][6] aka **FCS**.  

I also experimented quite a bit with a custom tokeniser that allowed the keywords and other syntax elements to be highlighted as tooltips:  

{{< figure src="https://lh4.googleusercontent.com/-_sHdxrBme-I/UvkMM1WVHHI/AAAAAAAAB0M/OtF0vnUdilI/w840-h344-no/keyword+tooltip.png" >}}  
At the time the F# binding was going through quite a turbulent change to the source code, and the combination of trying to develop FSharp.Compiler.Services and also refactoring the existing code meant that I decided to put this on hold for the time being.  I hope to resurrected this work one day, I can see that keyword highlighting would be very useful to beginners, especially if a detailed description was available similar to what is available on MSDN.  It also addresses some bugs that are still outstanding so it would be nice to get finished.  

Over the last couple of months a lot of the common code around parsing of long identifiers has been refactored so that is can be shared with other tools.  In the F# Binding we also have an emacs plugin which uses this shared code.  We have also created a shared language agent which makes using these language services even easier still!  
- - -
### Swat all the bugs!  
Anyone remember [Picnic paranoia][1]?  {{< figure class="img-left" src="http://i20.photobucket.com/albums/b223/NeoZeedeater/PicnicParanoiaTI.png" title="Swat all the bugs" >}}  

A shed load of bugs have also been swatted.  Using the F# addin is no longer like ignoring the **Unstable Structure sign** and hoping things don't crumble under your feet.  Admittedly there is still work to be done but some of the recent refactoring make it even easier to contribute to either new features or fixing bugs.  We've also added the F# binding to [up-for-grabs.net][8] and added the [up-for-grabs][3] issues as a starting point for anyone thinking about to helping out.  There's been a lot of blood sweat and tears and its consumed quite a lot of my spare time but I hope you all enjoy using the new F# addin features.  

Lastly a big thank you to everyone who has helped contribute over the last year, keep the contributions coming in!   

That's all for now, see you next time!

{{< EssentialListening
    "|http://upload.wikimedia.org/wikipedia/en/6/69/Silence_Followed_By_a_Deafening_Roar_album.jpg|Paul Gilbert - Silence Followed By a Deafening Roar"
    "|http://upload.wikimedia.org/wikipedia/en/1/1c/Get_Out_of_My_Yard_Artwork.jpg|Paul Gilbert - Get out of my yard" >}}

[1]: http://www.videogamehouse.net/picnicparanoia.html
[2]: http://monodevelop.com
[3]: https://github.com/fsharp/fsharpbinding/issues?direction=desc&labels=up-for-grabs&sort=created&state=open
[4]: http://fsharp.github.io/fsharpbinding/
[5]: https://xamarin.com/studio
[6]: http://fsharp.github.io/FSharp.Compiler.Service/
[7]: http://msdn.microsoft.com/en-us/library/dd233154.aspx
[8]: http://up-for-grabs.net/#/tags/f#
[9]: http://v2matveev.blogspot.co.uk/2010/07/tricky-late-binding-operators.html?utm_source=feedburner&utm_medium=feed&utm_campaign=Feed:+OccasionalNotes+(Occasional+notes)

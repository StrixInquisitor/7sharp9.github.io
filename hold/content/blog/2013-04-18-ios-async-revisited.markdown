---
author: 7sharp9
layout: post
title: "iOS async revisited"
date: 2013-04-18
comments: true
categories: [Programming]
tags: [F#, C#, iOS, programming, async, await]
---
In this post weare going to look as async again, but from the perspective of F#.  

###Xamarin Evolve 2013  
{% img http://blog.xamarin.com/wp-content/uploads/2013/04/Screen-Shot-2013-04-03-at-11.07.21-AM.png Xamarin Evolve 2013 %}  
I have been watching the Xamarin Evolve conference this week and it was good to see Miguel announce full support for F#.  Those that follow me on twitter etc, will know that I have been doing F# for quite a while in MonoDevelop and Xamarin Studio.  The new support currently entails some new project templates so that you can easily create epic new F# Apps without having to refer to my blog.  While its sad that my content now falls into the archives its nice to get official support announced in such a grand fashion.  <!-- more -->

### F# Async
Kudos to Miguel for covering some history of C#'s async feature right back down to its F# heritage too, which appeared in 2007, thanks to the work of Don Syme and the F# team.  You can read more about that on [Don Syme's blog][1] or have a look at the research paper [here][7].

So the highly anticipated async await model in C# that's just gone beta in Xamarin addin channel?  We've had it for ages in F#!  In fact, I suspect you will have been able to use it for quite some time, even before I started hacking together support for F# in iOS!  Anyway, that's enough of the smugness :-) lets get on and see what it looks like using some of the code from the previous post as a reference.  

Ill include the C# version first so that you can see the difference rather than having to open my last post.

{% codeblock lang:csharp %}
// Asynchronous HTTP request
public async void HttpSample ()
{
    Application.Busy ();
    var request = WebRequest.Create (Application.WisdomUrl);

    //async await version
    try{
        var response = await request.GetResponseAsync();
        Application.Done ();
        ad.RenderRssStream(response.GetResponseStream());
    } catch {
        // Error
    }
}
{% endcodeblock %}

One of the advantages of the F# Async model is it's composable nature and controllability.  The key to F# async is that its defined with F#'s [computation expression][2] syntax:

>Computation expressions in F# provide a convenient syntax for writing computations that can be sequenced and combined using control flow constructs and bindings.

There are several built in workflows: [Sequences][3], [Asynchronous Workflows][4], and [Query Expressions][5].  Whenever you use a computation expression it is as follows:- `builder-name { expression }`.  With that tiny bit of background, lets look at the corresponding F# async code:  
{% codeblock lang:fsharp %}
member x.HttpSample() =
    Application.Busy() 
    let request = WebRequest.Create(Application.WisdomUrl )
    
    //F# async version
    async {try let! response = request.AsyncGetResponse()
               Application.Done()
               ad(response.GetResponseStream())
           with ex -> () } |> Async.Start
{% endcodeblock %}

You can see there is quite a similarity between this snippet and the C# one and you should be able to figure out what's happening given the knowledge from the previous post.  

One of the first things you will notice the builder - `async { ...`, followed by the `let!` statement.  You can think of the `let!` as the C# equivalent of await.  `let!` starts the computation `request.AsyncGetResponse()`, and then the thread is suspended until the result is available, at this point execution continues to the next statment, which in this case is `Application.Done()`.  

Those of you comparing the difference will notice that in the C# version after `Application.Done();` we call `ad.RenderRssStream(response.GetResponseStream())` but in the F# version we simply call `ad(response.GetResponseStream())`.  If we take a quick look at the constructors for the types that hold these methods I can show you the difference a bit better:  

The C# version looks like this:    
{% codeblock lang:csharp  %}
public class DotNet 
{
	AppDelegate ad;

	public DotNet (AppDelegate ad)
	{
		this.ad = ad;
	}
}
{% endcodeblock %}

The F# one I can show on a single line:  
{% codeblock lang:fsharp  %}
type DotNet(ad: Stream -> unit) =
{% endcodeblock %}

The main difference is that The C# version has the entire `AppDelegate` class is passed in, whereas the F# version just takes a function with the signature `Stream -> unit`.  In fact the F# version doesn't even need to be placed inside a type like the C# version, we can use a [module][8], again Ill quote from MSDN: 

>In the context of the F# language, a module is a grouping of F# code, such as values, types, and function values, in an F# program. Grouping code in modules helps keep related code together and helps avoid name conflicts in your program.

### F# Modules  
{% codeblock lang:fsharp %}
module DotNet' =
    let HttpSample(ad) =
            Application.Busy() 
            let request = WebRequest.Create(Application.WisdomUrl )
            
            //F# async version
            async {try let! response = request.AsyncGetResponse()
                       Application.Done()
                       ad(response.GetResponseStream())
                   with ex -> () } |> Async.Start
{% endcodeblock %}

When we want to call this code we can open the module like you would a namespace:

{% codeblock lang:fsharp %}
open DotNet
HttpSample(ad)
{% endcodeblock %}

Or access it fully qualified by including the `module` name:  
{% codeblock lang:fsharp %}
DotNet.HttpSample(ad)
{% endcodeblock %}

How would this code look from the context of this sample application?

Here is a snipped from the AppDelegate code which makes use of this module

{% codeblock lang:fsharp %}
// This method is invoked when the application has loaded its UI and its ready to run
override x.FinishedLaunching (app:UIApplication, options:NSDictionary) =
    x.window.AddSubview (x.navigationController.View)
    x.button1.TouchDown.Add 
        (fun _ ->  if not UIApplication.SharedApplication.NetworkActivityIndicatorVisible then           
                       match x.stack.SelectedRow() with
                       | 0 -> DotNet.HttpSample x.RenderRssStream
                       | 1 -> DotNet.HttpSecureSample x.RenderStream
                       | _ -> (new Cocoa(x.RenderRssStream)).HttpSample() |> ignore )    
    TableViewSelector.Configure (x.stack, [|"http  - WebRequest"
                                            "https - WebRequest"
                                            "http  - NSUrlConnection" |] )                    
    x.window.MakeKeyAndVisible()
    true
{% endcodeblock %}

There are a few departures from the C# sample code which Ill include below now:

{% codeblock lang:csharp%}
// This method is invoked when the application has loaded its UI and its ready to run
public override bool FinishedLaunching (UIApplication app, NSDictionary options)
{
  window.AddSubview (navigationController.View);

  button1.TouchDown += Button1TouchDown;
  TableViewSelector.Configure (this.stack, new string [] {
    "http  - WebRequest",
    "https - WebRequest",
    "http  - NSUrlConnection"
  });

  window.MakeKeyAndVisible ();

  return true;
}

void Button1TouchDown (object sender, EventArgs e)
{
  // Do not queue more than one request
  if (UIApplication.SharedApplication.NetworkActivityIndicatorVisible)
    return;

  switch (stack.SelectedRow ()){
  case 0:
    new DotNet (this).HttpSample ();
    break;

  case 1:
    new DotNet (this).HttpSecureSample ();
    break;

  case 2:
    new Cocoa (this).HttpSample ();
    break;
  }
}
{% endcodeblock %}

Firstly we are using an lambda expression for the event handler via the Add method rather than the += handler which we use in C#.  We are also using F#'s awesome [pattern matching][6] feature on the results of `x.stack.SelectedRow()`.  This allows you to encode complex logic and also have the compiler assist you by catching non covered cases.  

I'm going to leave it there for now as I don't want to bombard any newcomers with tons of new F# features, and I also don't want to teach any of my regular F# followers how to suck eggs.  If anyone has a preference for more in depth comparisons to the C# version then let me know then I can tailor that into further posts on the subject.  

Until next time!

* * *
####Essential listening:  
*   Jason Becker - Perpetual Burn
*   Megadeth - Rust In Peace
*   Xantrix - Scourge  

    {% img http://upload.wikimedia.org/wikipedia/en/b/b4/PerpetualBurn.jpg 125  Jason Becker - Perpetual Burn %}{% img http://upload.wikimedia.org/wikipedia/en/d/dc/Megadeth-RustInPeace.jpg 125 Megadeth - Rust In Peace %}{% img http://www.metal-archives.com/images/4/6/3/9/4639.jpg?3304 125%}

[1]: http://blogs.msdn.com/b/dsyme/archive/2013/03/24/asynchronous-programming-from-f-to-python.aspx
[2]: http://msdn.microsoft.com/en-gb/library/dd233182.aspx
[3]: http://msdn.microsoft.com/en-gb/library/dd233209.aspx
[4]: http://msdn.microsoft.com/en-gb/library/dd233250.aspx
[5]: http://msdn.microsoft.com/en-gb/library/hh225374.aspx
[6]: http://msdn.microsoft.com/en-gb/library/dd547125.aspx
[7]: http://research.microsoft.com/apps/pubs/default.aspx?id=147194
[8]: http://msdn.microsoft.com/en-gb/library/dd233221.aspx
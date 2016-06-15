---
author: 7sharp9
layout: post
title: "A little bit of iOS async"
date: 2013-04-16
comments: true
categories: [programming]
tags: [C#, iOS, programming, async, await]
description: ""
type: post
---
I was going to title this post as 'Now for something completely different' but felt that a little bit too [Pythonesque][8], and when I thought about it a bit it isn't really completely just slightly different, namely C# rather than my usual F# posts.  <!-- more -->

Right, enough of the waffling, this post is a little tour into the relatively unknown area of async on iOS.  Xamarin announced the alpha preview of async await on March 11th this year (2013).  There are a couple of blog post floating around on the net if you look around, Rodrigo Kumpera posted a small example [here][1].  

I'm no stranger to async, I have spent a great deal of time over the years debugging and refining [IAsyncResult][2] style procedures and found Jeffrey Richter and Joe Duffy's books below to be an excellent reference for those interested.

<a href="http://www.amazon.com/gp/product/0735667454/ref=as_li_ss_il?ie=UTF8&camp=1789&creative=390957&creativeASIN=0735667454&linkCode=as2&tag=blacguitandge-20"><img border="0" src="http://ws.assoc-amazon.com/widgets/q?_encoding=UTF8&ASIN=0735667454&Format=_SL160_&ID=AsinImage&MarketPlace=US&ServiceVersion=20070822&WS=1&tag=blacguitandge-20" ></a><img src="http://www.assoc-amazon.com/e/ir?t=blacguitandge-20&l=as2&o=1&a=0735667454" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" /><a href="http://www.amazon.com/gp/product/032143482X/ref=as_li_ss_il?ie=UTF8&camp=1789&creative=390957&creativeASIN=032143482X&linkCode=as2&tag=blacguitandge-20"><img border="0" src="http://ws.assoc-amazon.com/widgets/q?_encoding=UTF8&ASIN=032143482X&Format=_SL160_&ID=AsinImage&MarketPlace=US&ServiceVersion=20070822&WS=1&tag=blacguitandge-20" ></a><img src="http://www.assoc-amazon.com/e/ir?t=blacguitandge-20&l=as2&o=1&a=032143482X" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" />

If you looking for a book on TPL/parallel programming then Parallel Programming with Microsoft .NET by Stephen Toub et al. is also a good read.  

<a href="http://www.amazon.com/gp/product/0735651590/ref=as_li_ss_il?ie=UTF8&camp=1789&creative=390957&creativeASIN=0735651590&linkCode=as2&tag=blacguitandge-20"><img border="0" src="http://ws.assoc-amazon.com/widgets/q?_encoding=UTF8&ASIN=0735651590&Format=_SL160_&ID=AsinImage&MarketPlace=US&ServiceVersion=20070822&WS=1&tag=blacguitandge-20" ></a><img src="http://www.assoc-amazon.com/e/ir?t=blacguitandge-20&l=as2&o=1&a=0735651590" width="1" height="1" border="0" alt="" style="border:none !important; margin:0px !important;" />

Anyway, enough of the book references lets look at an iOS example of async using the [MonoTouch samples][3] from Xamarin as a reference.

We are going to use the [HttpSample][7].  Its a relatively simple example that has several buttons which trigger an asynchronous request for data, when the data is returned it's simply rendered onto the screen.  

Lets look at the first asynchronous call in the [DotNet.cs][4] file, the `HttpSample`  method:

```csharp
// Asynchronous HTTP request
public void HttpSample ()
{
	Application.Busy ();
	var request = WebRequest.Create (Application.WisdomUrl);
	request.BeginGetResponse (FeedDownloaded, request);
}

// Invoked when we get the stream back from the twitter feed
// We parse the RSS feed and push the data into a table.
void FeedDownloaded (IAsyncResult result)
{
	Application.Done ();
	var request = result.AsyncState as HttpWebRequest;
	
	try {
    		var response = request.EndGetResponse (result);
		    ad.RenderRssStream (response.GetResponseStream ());
	} catch {
		// Error				
	}
}
```

To convert this to the async await style all we have to do is use the async and await keywords (surprise surprise!), and in this instance use the new **Async** suffixed methods on the `WebRequest` class.  

```csharp
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
```

As you can see the `request.BeginGetResponse` method has been changed to `await request.GetResponseAsync()` and the callback method `FeedDownloaded` which was passed into the `BeginGetRespose` method has now been assimilated into the `HttpSample` method.  The await keyword is acting as a wait point or suspension while the asynchronous method completes.  As soon the asynchronous call completes then processing continues to the line below, much in the same way that the callback code is executed in the IAsyncResult version.  For an in depth description then you can take a look at the [MSDN documentation][6] on the subject.

You could add a WebException to the catch block here, you would expect on situations like network outage which would result in DNS lookup failures from the async call.  

We can look at the next asynchronous method too the 'HttpSecureSample' method

```csharp
// Asynchornous HTTPS request
public void HttpSecureSample ()
{
	var https = (HttpWebRequest) WebRequest.Create ("https://gmail.com");

	// To not depend on the root certficates, we will accept any certificates:
	ServicePointManager.ServerCertificateValidationCallback = (sender, cert, chain, ssl) =>  true;

	https.BeginGetResponse (GmailDownloaded, https);
}

// This sample just gets the result from calling https://gmail.com, an HTTPS secure 
// connection, we do not attempt to parse the output, but merely dump it as text
void GmailDownloaded (IAsyncResult result)
{
	Application.Done ();
	var request = result.AsyncState as HttpWebRequest;

	try {
    		var response = request.EndGetResponse (result);
		    ad.RenderStream (response.GetResponseStream ());
	} catch {
		// Error
	}
}
```

Lets have a look at the async await version of that:

```csharp
// Asynchornous HTTPS request
public async void HttpSecureSample ()
{
	var https = (HttpWebRequest) WebRequest.Create ("https://gmail.com");

	// To not depend on the root certficates, we will accept any certificates:
	ServicePointManager.ServerCertificateValidationCallback = (sender, cert, chain, ssl) =>  true;

    try {
			var response = await https.GetResponseAsync();
			Application.Done ();
			ad.RenderRssStream (response.GetResponseStream ());
	} catch {
		// Error
	}
}
```

In this situation there was also a version of the IAsyncResult Begin/End pattern that had been converted to async - `https.GetResponseAsync()` and we also had to add the `async` keyword to the `HttpSecureSample` method.  

For the situations where there is no **Async** suffixed method available you can build your own using the [Task.Factory.FromAsync][5] methods.  I wont go into the details of that here but if anyone wants any information on that then just give me a shout and I can revisit in in a future post.  

Ah yes, I almost forgot, there are some common pitfalls of using async await and Tomas Petricek posted a good compilation of them the other day: [C# async gotchas][9].

Until next time!

* * *
#### Essential listening:  
*   Paul Gilbert - Silence Followed By A Deafening Roar  
*   FooFighters - The Colour And The Shape  

{{< img-fit
	2u "http://upload.wikimedia.org/wikipedia/en/6/69/Silence_Followed_By_a_Deafening_Roar_album.jpg" "Paul Gilbert - Silence Followed By A Deafening Roar"
	2u "http://upload.wikimedia.org/wikipedia/en/0/0d/FooFighters-TheColourAndTheShape.jpg" "FooFighters - The Colour And The Shape" "" "" >}}


[1]: http://blog.xamarin.com/brave-new-async-mobile-world/
[2]: http://msdn.microsoft.com/en-GB/library/system.iasyncresult.aspx
[3]: https://github.com/xamarin/monotouch-samples
[4]: https://github.com/xamarin/monotouch-samples/blob/master/HttpClient/DotNet.cs
[5]: http://msdn.microsoft.com/en-us/library/system.threading.tasks.taskfactory.fromasync.aspx
[6]: http://msdn.microsoft.com/en-gb/library/vstudio/hh191443.aspx
[7]: https://github.com/xamarin/monotouch-samples/tree/master/HttpClient
[8]: http://en.wiktionary.org/wiki/Pythonesque
[9]: http://tomasp.net/blog/csharp-async-gotchas.aspx

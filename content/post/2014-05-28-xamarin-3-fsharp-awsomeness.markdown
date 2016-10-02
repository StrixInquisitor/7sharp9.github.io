---
author: 7sharp9
layout: post
title: "Xamarin 3 F# Awesomeness"
date: 2014-05-28
comments: true
categories: [programming]
tags: [FSharp, Xamarin, tools, Xamarin.Forms]
description: ""
type: post
---
With the release of Xamarin 3 there is a swathe of new features to the platform, but obviously the most important one is obviously F# support is now included by default in Xamarin Studio so there is no escape from the awesomeness of F#!  

So what else have we got, well loads of other goodies too like Xamarin Designer for iOS, Xamarin.Forms, Major IDE enhancements, Improved code sharing with PCL and Shared projects, and BCL Documentation.  You can see the release blog post here: [announcing Xamarin 3][1]  

There are tons of things I could show you, but for this post lets have a quick look at Xamarin.Forms    
<!-- more -->  
## Xamarin.Forms
So what is Xamarin.Forms?  Well to paraphrase the Xamarin blurb a little bit:

>Build native UIs for iOS, Android and Windows Phone
from a single, shared codebase.  Xamarin.Forms pages represent single screens within an app.  

>Pages contain layouts, buttons, labels, lists, and other common controls. Connect these controls to shared backend code and you get fully native iOS, Android, and Windows Phone apps built entirely with shared code.  

So as this just a whistle-stop post I'll quickly show you the code.:   

```fsharp
let profilePage = 
   ContentPage(Title="Profile",
               Icon="Profile.png",
               Content=StackLayout.Create(
                           [ Entry (Placeholder="Username") 
                             Entry (Placeholder="Password", IsPassword=true)
                             Button (Text="Login",
                                     TextColor=Color.White, 
                                     BackgroundColor=Color.FromHex "77D065") ], 
                             spacing=20.0,
                             padding=thickness 50.0,
                             verticalOptions=LayoutOptions.Center))
                        
let settingsPage = ContentPage(Title="Settings", Icon="Settings.png")

let mainPage = TabbedPage.create [profilePage;settingsPage]
```

The result is a nice cross platform UI thats easy to build and maintain, sweet!  

{{< figure src="https://xamarin.com/content/images/pages/forms/example-app.png" >}}

You can see by the code thats there are several components at play but it's very easy to see how the code relates to the UI layout.  A stack layout with two `Entry` types followed by a `Button`.  You can also see at the bottom that there is a `TabbedPage` comprising of the `profilePage` and the `settingPage`.  

Nice and easy!  

OK, thats it for now, but next time I'll be delving in a little deeper and showing you exactly how to build a PCL backed F# app over multiple platforms.  

Until next time!  

{{< EssentialListening
    "|http://upload.wikimedia.org/wikipedia/en/1/1c/Iron_Maiden_-_Powerslave.jpg|Iron Maiden - Powerslave"
    "|http://upload.wikimedia.org/wikipedia/en/6/6b/Electric_Sea.jpg|Buckethead - Electric sea" >}}

[1]: http://blog.xamarin.com/announcing-xamarin-3/
[2]: https://xamarin.com/forms

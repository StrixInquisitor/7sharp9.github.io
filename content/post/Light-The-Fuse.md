+++
author = "7sharp9"
categories = ["programming"]
tags = ["FSharp", "Fuse", "Fable", "Transpiling", "JavaScript"]
date = "2016-06-03T11:21:57+01:00"
description = "Combining Fable and Fuse"
title = "Light The Fuse"
type = "post"
images = ["http://7sharpnine.com/img/fable-fuse/fable-fuse-tc.png?5"]
+++
In this post Im going to be introducing something new and exciting the combination of F#, Fable and Fuse.  If I was American I might say this was the holy trinity of awesomeness!<!--more-->

This is the start of an exciting new series on using F# as a transpiling language to "light the Fuse" (pun intended) on new platform and opportunities for F#.  

OK, so now for the introductions...
- - -
# What is Fable

Put simply Fable is a transpiler for F# that turn F# into JavaScript.  Lets face it JavaScript is really pervasive but not everyone wants to write it.  You can read about how Fable works [here][2]

# What is fuse?

Taken from their site this is what fuse is in a nutshell: 

## FUSE IS FOR MOBILE APP DESIGNERS AND DEVELOPERS

Create and update the look and feel for native apps in real time
on multiple devices simultaneously. 

Fuse is a set of tools (currently in beta) that makes design and developing native mobile apps for iOS and Android fast, easy and fun. Fuse is free, and we’re actively working towards making it Open Source as well.

Fuse introduces UX Markup - a XML-based language for creating truly native, data-driven, responsive, smoothly animated and highly interactive experiences, while sharing most of the code between iOS and Android. UX is easy to learn and incredibly powerful.

Fuse is fast. Based on Uno, a language which compiles down to pure C++ code, and seamlessly interops with Objective-C (iOS) and Java (Android) where needed. The UI is rendered using native platform controls, OpenGL or a combination (best of both worlds).

For business logic, Fuse runs JavaScript on a separate thread on both iOS and Android, so your UI is fast and responsive no matter what it is doing. Fuse lets you call seamlessly into C++, Java and Objective-C libraries through Uno when you need it.  

You can read about Fuse in more depth on their site [here][1], you really should take a look.  
- - -
# Introducing **Fable |> Fuse**

Fable-Fuse is a set of packages that allows you use power of F# with [Fuse][1].  

First of all lets have a little look at some declarative UI with Fuse.

This is taken from the original Fuse Sample titled ***Parsing JSON fetched over HTTP***   which is located [here][3]  

{{< figure src="https://res.cloudinary.com/fusetools/image/upload/w_450%2Ch_450%2Cdpr_1.0%2Cc_limit/examples/media/4343ca40291fab07d70ef87254204bed_http-json-example.webp" >}}
```
<App Theme="Basic" Background="#eee">
    <DockPanel>
        <StatusBarBackground Dock="Top" />
        <BottomBarBackground Dock="Bottom" />
        <ScrollView>
            <Grid ColumnCount="2">
                <JavaScript>
                    var Observable = require("FuseJS/Observable");

                    var data = Observable();

                    fetch('http://az664292.vo.msecnd.net/files/ZjPdBhWNdPRMI4qK-colors.json')
                        .then(function(response) { return response.json(); })
                        .then(function(responseObject) { data.value = responseObject; });

                    module.exports = {
                        data: data
                    };
                </JavaScript>
                <Each Items="{data.colorsArray}">
                    <DockPanel Height="120" Margin="10,0">
                        <Panel DockPanel.Dock="Top" Margin="10" Height="30">
                            <Rectangle Layer="Background" CornerRadius="10" Fill="#fff"/>
                            <Text Value="{colorName}" TextAlignment="Center" Alignment="Center" />
                        </Panel>

                        <Rectangle Layer="Background" CornerRadius="10" Fill="{hexValue}"/>
                    </DockPanel>
                </Each>
            </Grid>
        </ScrollView>
    </DockPanel>
</App>
```

You can see the UI mark-up is concise and easy to follow, also notice the JavaScript element which can be in-line, as shown here, or placed in a separate file.  We will place this in a separate file so that we can transpile from F# using [Fable][2].

```fsharp
namespace Program
open Fable.Core
open Fuse
open Fable.Import
open Fable.Import.Fetch

module HttpJson =
    let data = Observable.create()
    promise {
        let! req = GlobalFetch.fetch (Url "http://az664292.vo.msecnd.net/files/ZjPdBhWNdPRMI4qK-colors.json")
        let! json = req.json ()
        do (data.value <- json) } |> ignore
```

You can see here there is a custom computation expression that allows you to use JavaScrip promises, you could also use a pipeline oriented definition too, like this:

```fsharp
GlobalFetch.fetch (Url "http://az664292.vo.msecnd.net/files/ZjPdBhWNdPRMI4qK-colors.json")
|> Promise.success (fun resp -> resp.json())                                                                          
|> Promise.success (fun json -> data.value <- json) 
|> ignore
```

Or if you really really wanted to you could integrate this into an F# async with a little helper type extension:
```fsharp
module AsyncExtensions =    
    type Microsoft.FSharp.Control.AsyncBuilder with
        member x.Bind(p, f) = 
            async.Bind (Async.AwaitPromise(p), f)
```

This would allow you to use a promise with an ordinary `let!`
```fsharp
async {
    let! req = GlobalFetch.fetch "http://az664292.vo.msecnd.net/files/ZjPdBhWNdPRMI4qK-colors.json"
    let! json = req.json ()
    do (data.value <- json) } |> Async.Start 
```

Anyway I digress, needless to say there are various options with promises and how to handle them with **Fable |> Fuse**.  

{{< figure src="https://i.imgur.com/P4YcEi7.png" >}}

What's more because Fuse and Fable are real-time you can edit the UX definitions and it the application in real-time across multiple devices!

{{< figure src="/img/fable-fuse/realtime.gif" >}}
- - -
## So how do I get started with **Fable |> Fuse** ?

Well you'll have to hold your horses, I was so excited to share this introductory post I haven't wrote that part yet.  The package I'm working on is still private while I finalise things and make it really easy and friendly to create applications with **Fable |> Fuse**.  

Stay tuned as there will be more in this series next week as I discuss the more technical aspects and how to create your first **Fable |> Fuse** application.  

If there is enough interest I will also live stream this on my [livecoding.tv channel][6], please subscribe.  

A really big thanks to Alfonso Garcia-Caro ([@alfonsogcnunez][5]) creator of Fable for answering all my annoying questions.  And Lars Thomas Denstad ([@cocporn][4]) for help creating the Fuse API bindings for Fable.   

Until next time!

[1]: https://www.fusetools.com/
[2]: http://fsprojects.github.io/Fable/
[3]: https://www.fusetools.com/examples/http-json
[4]: https://twitter.com/COCPORN
[5]: https://twitter.com/alfonsogcnunez
[6]: https://www.livecoding.tv/7sharp9/

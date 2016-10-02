---
author: 7sharp9
date: '2011-01-28'
layout: post
slug: sockets-and-bockets-part-4
status: publish
title: Sockets and Bockets 4
wordpress_id: '180'
comments: true
categories:
- programming
series: [socketsandbockets]
tags:
- FSharp
- GC
- Memory fragmentation
- Multithreading
- Performance
- SAEA
- Sockets
- YourKit
description: ""
type: post
---

## Welcome to part 4

If you were looking forward to some exciting new F# code this time your going
to be disappointed, however if you are like me and like looking at graphs and
stats and digging in deeper into the code then your going to enjoy this, lets
get started...<!-- more -->

I set up a 5 minute test with 50 clients connecting to the server with a 15ms
interval between each one.  Once connected each client receives a 128 byte
message from the server every 100ms so this will be a 500 messages per second
test.  I am going to be using an excellent product called [YourKit Profilerfor .NET](http://bit.ly/e4ToaO ) it can do both memory and CPU profiling as
well as displaying telemetry for things like thread count, stack contents,
memory allocations etc.  It can be configured to be a lot less intrusive than
a lot of other profilers and I have had a lot of success using it.  You can
download a demo from their site using the link above.  I will be doing some
other articles on using profiling and analysis tools later on so stay tuned
for those too.  All of the graphs and information gathered in this post come
from YourKits output during CPU and memory profiling.

Before we start here's a reminder of what the client code looks like, this is
a simple test client using Brian's code as mentioned in
[Part1](http://7sharpnine.com/posts/sockets-and-bockets-1/) I have highlighted the lines that
have changed below.

```fsharp   
    open System.Net
    open System.Net.Sockets  
    let quoteSize = 128  
    type System.Net.Sockets.TcpClient with
      member client.AsyncConnect(server, port, clientIndex) =
        Async.FromBeginEnd(server, port,(client.BeginConnect : IPAddress * int * _ * _ -> _), client.EndConnect)  
    let clientRequestQuoteStream (clientIndex, server, port:int) =
      async {
        let client = new System.Net.Sockets.TcpClient()
        do!  client.AsyncConnect(server,port, clientIndex)
        let stream = client.GetStream()
        let! header = stream.AsyncRead 1 // read header
        while true do
          let! bytes = stream.AsyncRead quoteSize
          if Array.length bytes <> quoteSize then
            printfn "client incorrect checksum"
      }  
    let myLock = new obj()  
    let clientAsync clientIndex =
      async {
        do! Async.Sleep(clientIndex*15)
        if clientIndex % 10 = 0 then
          lock myLock (fun() -> printfn "%d clients..." clientIndex)
        try
          do! clientRequestQuoteStream (clientIndex, IPAddress.Loopback, 10003)
        with e ->
          printfn "CLIENT %d ERROR: %A" clientIndex e
          //raise e
      }  
    Async.Parallel [ for i in 1 .. 50 -> clientAsync i ]
      |> Async.Ignore
      |> Async.Start
    System.Console.ReadKey() |> ignore
```

## CPU and threading performance

First of all lets look at the CPU results from the _IAsync_ pattern:

{{< figure src="https://lh5.googleusercontent.com/-3H8-TiiB-VI/TwYhL2mvYsI/AAAAAAAABQI/z8dmiHBvTJE/mcnamara-cpu1.png" >}}

Heres the same run from the SAEA pattern:

{{< figure src="https://lh3.googleusercontent.com/-yGB2zdE3kGM/TwYhLPrUmLI/AAAAAAAABP8/Y6CIMWOi4Gk/Bocket-cpu2.png" >}}

You can see that both the number of threads and the amount of CPU is a quite a
lot less in the SAEA pattern.  The spike at the beginning is the allocation of
buffers for the BocketPool.

Now lets move on to memory and garbage collection.

## Memory Allocation

Here's a graph of the heap and process memory allocation in the **IAsync**
pattern, green is generation 0, blue is generation 1 and orange is the large
object heap.  There's also red for generation 2 but the results are behind the
others and they are only small 0,2 MB peaks at 5 to 15 second intervals.

{{< figure src="https://lh3.googleusercontent.com/-PalohQxAkOg/TwYhL6hR5JI/AAAAAAAABQQ/NtCLYc43OZc/mcnamara-mem1.png" >}}

Heres the same but for the SAEA pattern, there are red peaks every 10- 20
second intervals of 0.2MB hidden under the others.

{{< figure src="https://lh3.googleusercontent.com/-Nadz1nXQ7lg/TwYhLBXMvcI/AAAAAAAABQM/xCfuXkzkekM/Bocket-mem1.png" >}}

As you can see the heap memory is around half the size and the process memory is 15MB less.

## Memory Hotspots

Finally here's a couple of screen shot of the hot spots for memory allocations
in both implementations

{{< figure src="https://lh4.googleusercontent.com/-n3QWLgvNjq8/TwYhLX8cIZI/AAAAAAAABQA/LkeRno775Ew/s800/IAsync-hot.png" title="IAsync" >}}

{{< figure src="https://lh5.googleusercontent.com/-PSX_YUfxkgU/TwYhMr40DTI/AAAAAAAABQY/D8bgLS6kNwc/s800/SAEA-hot.png" title="SAEA" >}}

You can clearly the **IAsync** allocations are not present in the SAEA
implementation and there are 310,188 of them, that's 27% of the total garbage!

## Final thoughts

The SAEA pattern definitely cuts down on memory and CPU usage, yes it adds a
lot of complexity but if your application is dealing with a very high volume
of traffic or clients and you need optimal performance then I think its the
way to go.

The optimisations don't stop there either, if you think about it the receive
Bocketpool is not even used here, if we collapsed all of the BocketPools into
a single contiguous store then we would use even less resources, this means we
could support even more clients or throughput.  In a typical high volume
scenario you are looking at doubling your throughput or number of client
connections.

There's definitely a lot more or interesting things to explore in this area.

As usual any comments are welcome.

See you next time...
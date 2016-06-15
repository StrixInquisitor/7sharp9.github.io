---
author: 7sharp9
date: '2011-12-11'
layout: post
slug: fixing-a-hole
status: publish
title: Fixing a hole...
wordpress_id: '505'
comments: true
categories:
- programming
tags:
- Async Workflows
- F#
- fractureIO
- GC
- memory profiling
- Performance
- pipelets
description: ""
type: post
---
Due to popular demand... well, I had a couple of requests anyway :-) Heres
a post inspired by my recent encounters profiling some of the code in
[Fracture-IO](https://github.com/fractureio/fracture).  <!-- more -->I have recently been
profiling the code in fracture to remove any so called low hanging fruits.
During this time I also noticed an increase in memory allocation.  I
remembered I had recently been experimenting in a branch using pipelets as a
buffer between the send and receive stages in the Http Server, so I set up a
simple test to see if pipelets were contributing to the memory allocation
issues I was seeing.  Here's the simple iteration test code I used for the
memory profiling:

```fsharp
open System
open System.Diagnostics
open System.Threading
open Fracture.Pipelets  
let reverse (s:string) =
  String(s |> Seq.toArray |> Array.rev)  
let oneToSingleton a b f=
  let result = b |> f
  result |> Seq.singleton  
/// Total number to run through test cycle
let number = 100  
/// To Record when we are done
let counter = ref 0
let sw = new Stopwatch()
let countThis (a:String) =
  do Interlocked.Increment(counter) |> ignore
  if !counter % number = 0 then
    sw.Stop()
    printfn "Execution time: %A" sw.Elapsed.TotalMilliseconds
    printfn "Items input: %d" number
    printfn "Time per item: %A ms (Elapsed Time / Number of items)"
      (TimeSpan.FromTicks(sw.Elapsed.Ticks / int64 number).TotalMilliseconds)
    printfn "Press any key to repeat, press 'q' to exit."
    sw.Reset()
  counter |> Seq.singleton  
let OneToSeqRev a b =
  oneToSingleton a b reverse   
let generateCircularSeq (s) =
  let rec next () =
    seq {
      for element in s do
        yield element
      yield! next()
    }
  next()  
    let stage1 = new Pipelet<_,_>("Stage1", OneToSeqRev "1", Routers.roundRobin, number, -1)
    let stage2 = new Pipelet<_,_>("Stage2", OneToSeqRev "2", Routers.basicRouter, number, -1)
    let stage3 = new Pipelet<_,_>("Stage3", OneToSeqRev "3", Routers.basicRouter, number, -1)
    let stage4 = new Pipelet<_,_>("Stage4", OneToSeqRev "4", Routers.basicRouter, number, -1)
    let stage5 = new Pipelet<_,_>("Stage5", OneToSeqRev "5", Routers.basicRouter, number, -1)
    let stage6 = new Pipelet<_,_>("Stage6", OneToSeqRev "6", Routers.basicRouter, number, -1)
    let stage7 = new Pipelet<_,_>("Stage7", OneToSeqRev "7", Routers.basicRouter, number, -1)
    let stage8 = new Pipelet<_,_>("Stage8", OneToSeqRev "8", Routers.basicRouter, number, -1)
    let stage9 = new Pipelet<_,_>("Stage9", OneToSeqRev "9", Routers.basicRouter, number, -1)
    let stage10 = new Pipelet<_,_>("Stage10", OneToSeqRev "10", Routers.basicRouter, number, -1)
    let final = new Pipelet<_,_>("Final", countThis, Routers.basicRouter, number, -1)  
    let manyStages = [stage2;stage3;stage4;stage5;stage6;stage7;stage8;stage9;stage10]  
    oneToMany stage1 manyStages
    manyToOne manyStages final  
    System.AppDomain.CurrentDomain.UnhandledException |> Observable.add (fun x ->
      printfn "%A" (x.ExceptionObject :?> Exception); Console.ReadKey() |> ignore)  
    let circ = ["John"; "Paul"; "George"; "Ringo"; "Nord"; "Bert"] |> generateCircularSeq   
    let startoperations() =
      sw.Start()
      for str in circ |> Seq.take number
        do  str --> stage1
      printfn "Insert complete waiting for operation to complete."  
    printfn "Press any key to process %i items" number
    while not (Console.ReadKey().Key = ConsoleKey.Q) do
      startoperations()
```
Using process explorer from Mark Russinovich I watched the allocated memory
grow as the iterations progressed:

{{< figure src="https://lh4.googleusercontent.com/-VP1-Vo2VINU/TuS7yZFTTlI/AAAAAAAABNw/3ksn5vNXTtw/s400/leak.png" >}}

Theres definitely something leaking in there! So what can we do to find this?
Simple, we use a memory profiler.  There are several really good memory
profilers out there.  I have listed some of the best ones below:

  * [SciTech memory profiler](http://memprofiler.com/)
  * [RedGates ANTS Memory Profiler](http://www.red-gate.com/products/dotnet-development/ants-memory-profiler/)
  * [JetBrains dotTrace](http://www.jetbrains.com/profiler/)
  * [YourKit Profiler for .NET](http://www.yourkit.com/dotnet/features/index.jsp)
To demonstrate finding the leak I will be using [RedGates ANTS MemoryProfiler](http://www.red-gate.com/products/dotnet-development/ants-memory-
profiler/).

First of all we launch the profiler and set it up to profile the application,
this is just a simple case of browsing to the release folder and picking the
application so I won't bore with those trivial details here. Now that the
application is running we hit any key which caused the test application to
post 100 operations into the pipeline.  We want to create a baseline snapshot
of the memory allocation so we can see where our leak is.  To do this click
Take Memory Snapshot at the top right of the screen.  Next we hit any key
again in the test application, again causing it to post another 100 operations
into the pipeline.  Now we click Take Memory Snapshot again. Now we have a
snapshot of the difference between the two operations.  The summery screen is
shown below:

{{< figure src="https://lh4.googleusercontent.com/-N72POVbq0ZA/TuS7v0xBt1I/AAAAAAAABNQ/ExKfkJDCb50/s912/3%2Bsummary.png" >}}

From this screen you can see that there is 51.56KB of new memory allocated
since the last snapshot, and you can see some nice piecharts showing the
various allocations in G1, G2 etc.  On the right hand side of the pie chart
you can see that the largest classes are: object[], AsyncParamsAux,
Pipelets+loop@37-7<Unit, string,string>, and AsyncParams<Unit>.

Now if we click on Class List button we can investigate these further, heres
the Class List:

{{< figure src="https://lh4.googleusercontent.com/-UDU20rPk7ck/TuS7xvK_67I/AAAAAAAABNg/tnxTlpBYBJg/s912/4%2Bclass%2Blist.png" >}}

Here things start to get interesting.  If you click on the instance Diff (+/-)
column you can sort the list of classed by the differences to the last
snapshot.

Now looking at the results we have:

  * 300 more instances of AsyncBuilderImpl, AsyncParamsArgs, and AsyncParams
  * 200 more instances of Pipelets+loop@37-7<Unit, string, string>
  * 100 more instances of Pipelets+loop@37-7<Unit, string, FSharpRef<int>>

Is it a coincidence that we just pushed 100 operations through the pipeline?
I think not!

Now that we have a target for further inspection we can highlight the row for
the function **Pipelets+loop@37-7<Unit, string, FSharpRef<int>>>** and then
click on the icon that has three little blue boxes on it.  This will take us
to the instance List as shown below:

{{< figure src="https://lh6.googleusercontent.com/-csF589rWobQ/TuS7wzT6NWI/AAAAAAAABNY/TM95SUmaCEQ/s912/5%2Binstance%2Blist.png" >}}

I have sorted the instance list by the distance from the GC Root, you can see
there is a strange pattern emerging, the GC root distant increase by three
each time.  Now lets look at the Instance Retention graph for the first one
with a GC Root distance of 9, this is the icon on the right hand side of the
function name, it looks like a few rectangles joined up with a line:

{{< figure src="https://lh3.googleusercontent.com/-9orgQ4etsdI/TuS7yE-Dh7I/AAAAAAAABNs/0-dc3x8u-Mo/s525/first.png" >}}

The Pipelets+loop function is linked from the mailbox processor shown at the
top of the graph and flows into the Async infrastructure, and finally to the
loop function at the bottom.

Lets look at the next one, this has a GC Root distance of 12:

{{< figure src="https://lh3.googleusercontent.com/-j1eALRVz0kA/TuS7z0ycXDI/AAAAAAAABN8/SGPVpPQbSz0/s531/second.png" >}}

If you look carefully there is another pattern here, the field references
args, aux@, econt@ are repeated in the red boxes.  The functions look to be
quite similar too.  Lets look at the next one GC Root Distance  of 15:

{{< figure src="https://lh6.googleusercontent.com/-d3V1WkCktTM/TuS7zxVnizI/AAAAAAAABOA/wA5xlP-UTxA/s646/third.png" >}}

Looking at this we have a definite repeat of the functions and arguments,  if
we look down to GC Root at a depth of 60 we get this:

{{< figure src="https://lh3.googleusercontent.com/-X3v-_UfEkI4/TuS7xzETTHI/AAAAAAAABNk/js7xg3GTHIo/s640/60.png" >}}

So whats happening here is that there is a continuation that has been built
around the asynchronous calls that gets bigger and bigger on each iteration.

Now that we have identified the leak, lets look at the code and see whats
going on.  That would be the loop function in Pipelets:

```fsharp
let mailbox = MailboxProcessor.Start(fun inbox ->
  let rec loop routes = async {
    let! msg = inbox.Receive()
    match msg with
    | Payload(data) ->
      ss.Release() |> ignore
      try
        data |> transform |> router <| routes
        return! loop routes
      with //force loop resume on error
      | ex -> errors ex
          return! loop routes
    | Attach(stage) -> return! loop (stage::routes)
    | Detach(stage) -> return! loop (List.filter (fun x -> x <> stage) routes)
  }
  loop [])
```

Have a look at lines 9 and 12.  Can you guess whats wrong?

Well, to quote the [F# Teams blog](http://blogs.msdn.com/b/fsharpteam/archive/2011/07/08/tail-calls-in-fsharp.aspx):

> On the .NET platform, there are limitations on where tail calls may occur.
One restriction is that tail calls cannot be performed in try-catch or try-
finally blocks (neither in the body of the try nor in the catch or finally
handlers).

It goes on further to discuss another subtle issue with use bindings:

> use bindings implicitly generate a try-finally around the code that follows
them to ensure that the Dispose method is called on the bound value.  This
means that no calls following a use binding will be tail calls.

So all we have to do change the way the try catch block is formulated in that
section.  The most idiomatic way of dealing with this is to use the
[Async.Catch function](http://msdn.microsoft.com/en-us/library/ee353899.aspx)
which would result in code something like the following:

```fsharp
let mailbox = MailboxProcessor.Start(fun inbox ->
  let rec loop routes = async {
    let! msg = inbox.Receive()
    match msg with
    | Payload(data) ->
      ss.Release() |> ignore
      let result = async{data |> transform |> router <| routes} 
      |> Async.Catch 
      |> Async.RunSynchronously
      match result with
      | Choice1Of2() -> ()
      | Choice2Of2 exn -> errors exn
      return! loop routes
    | Attach(stage) -> return! loop (stage::routes)
    | Detach(stage) -> return! loop (List.filter (fun x -> x <> stage) routes)
  }
  loop [])
  ```

Alternatively you could move the entire try with section out to a more local
section thats not in the recursive async loop construct:

```fsharp
let computeAndRoute data routes =
  try
    data |> transform |> router <| routes
    Choice1Of2()
  with
  | ex -> Choice2Of2 ex  

let mailbox = MailboxProcessor.Start(fun inbox ->
  let rec loop routes = async {
    let! msg = inbox.Receive()
    match msg with
    | Payload(data) ->
      ss.Release() |> ignore
      match computeAndRoute data routes with
      | Choice2Of2 exn -> errors exn
      | _ -> ()
      return! loop routes
    | Attach(stage) -> return! loop (stage::routes)
    | Detach(stage) -> return! loop (List.filter (fun x -> x <> stage) routes)}
  loop [])
```

Anyway I hope that sheds a bit of light on how to spot where memory leaks are
stemming from, and also some of the little known and often forgotten caveats
with tail recursion.

Until next time...

**EDIT**: Just to make things a little bit clearer.  The memory leak here is caused by the async block being transformed into chains of continuation passing-style functions, and due to tail call elimination not being possible inside of the try catch blocks, the continuation grows and grows during each recursion.


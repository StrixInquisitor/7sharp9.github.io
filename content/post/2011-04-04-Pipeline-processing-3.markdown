---
author: 7sharp9
date: '2011-04-04'
layout: post
slug: pipeline-processing-3
status: publish
title: Pipeline processing 3
wordpress_id: '310'
comments: true
categories:
- programming
series: [pipelineprocessing]
tags:
- Async Workflows
- asynchronous
- Complexity
- FSharp
- Pipeline
- Sockets
- TPL
description: ""
type: post
---

Ok so I have been offline for a while now, what with starting a new financial contract in London and not having any broadband access for a while.  I have
been working on something, honest!

Since the last post I have been reflecting on the pipeline design and it had a distinct object orientated feel to it that I wasnt happy with, so I have
amended the structure of the code and come up with the following which simplifies in some areas and expands in others...<!-- more -->

```fsharp
module Pipeline
  open System.Collections.Concurrent

  [<Interface>]
  type IPipelineInput<'a> =
    abstract Insert: 'a -> unit
 
  [<Interface>]
  type IPipelineConnection<'a> =
    abstract Attach: IPipelineInput<'a> -> unit
    abstract Detach: IPipelineInput<'a> -> unit
 
  [<Interface>]
  type IPipeline<'a,'b> =
    inherit IPipelineConnection<'b>
    inherit IPipelineInput<'a>

  type PipelineStage<'a,'b>(processor, router: seq<IPipelineInput<'b>> * 'b -> seq<IPipelineInput<'b>>, ?overflow, ?capacity, ?blockingTime) =
    let processor = processor
    let router = router  
    let createBlockingCollection x =
        match x with
        | Some c -> new BlockingCollection<'a>(c:int)
        | None -> new BlockingCollection<'a>()  
    let buffer = createBlockingCollection capacity
    let routes = ref List.empty<IPipelineInput<'b>>
    let queuedOrRunning = ref false  
    let blocktime =
      match blockingTime with
      | Some b -> b
      | None -> 250  
    let consumerLoop = async {
      try
        let rec loop()=
          let item = ref Unchecked.defaultof<_>
          let taken = buffer.TryTake(item, blocktime)
          if taken then
              do !item
              |> processor
              |> Seq.iter (fun z ->
              (match !routes with
               | [] -> ()(*we cant route with no routes*)
               | _ -> do router (!routes, z) |> Seq.iter (fun r -> (r.Insert z ))) )
              loop()
          else ()(*exit nothing to consume in time limit*)
        loop()
      with e -> raise e
      }  
    member this.ClearRoutes = routes := []  
    interface IPipelineInput<'a> with
      member this.Insert payload =
        let added = buffer.TryAdd(payload, blocktime)
        if added then
          //begin consumer loop
          if not !queuedOrRunning then
            lock consumerLoop (fun() ->
            Async.Start(async {do! consumerLoop })
            queuedOrRunning := true)
          else()
        else
          //overflow here if function passed
          match overflow with
          | Some t ->  payload |> overflow.Value
          | None -> ()  
    interface IPipelineConnection<'b> with
      member this.Attach (stage) =
        let current = !routes
        routes := stage :: current  
      member this.Detach (stage) =
        let current = !routes
        routes := List.filter (fun el -> el <> stage) current  
    static member Attach (a:IPipelineConnection<_>) (b) =
      a.Attach b ;b  
    static member Detach (a: IPipelineConnection<_>) (b) =
      a.Detach b ;a  
    static member (++>) (a:IPipelineConnection<_>, b) =
      a.Attach (b) ;b  
    static member (-->) (a:IPipelineConnection<_>, b) =
      a.Detach b ;a  
    static member (<<--) (a:IPipelineInput<_>, b:'b) =
      a.Insert b  
    static member (-->>) (b,a:IPipelineInput<_>) =
      a.Insert b
```

### Summary.

I only want to summarise the code as I think its fairly straight forward to
see whats going on.

### Interfaces

We have two main interfaces defined **IPipelineInput<'a>** and
**IPipelineConnection<'a>, **as you can tell by the names they are involved
with connecting the pipeline together and getting information into the
pipeline.  Those two interfaces are merged together in the IPipeline<'a, 'b>
interface, this keeps a nice separation between connecting and inserting into
the pipeline, it also makes implementation easier and allows the interfaces to
be implemented in other areas of code that need to talk to or connect to a
pipeline.

### Internals

Inside the pipeline we have the bounded blocking queue which is implemented by
the BlockingCollection from TPL. This is used to store the pipeline payloads
that are waiting to be processed.

The consumerLoop function is recursive and continually tries to take items
from the blocking collection processing and routing each one to the next
pipeline stage.

The processor is a function that transforms from type 'a to type 'b.

The router is a function that takes a sequence of IPipelineInput<'b> and also
the payload 'b it returns a sequence of IPipelineInput<'b>.  What this
effectively means is that we can route by the connected stages (i.e. round
robin routing, multi-cast routing.)   Or we could route by payload contents
(i.e. if the payload contains a certain bytes sequence we could choose a
certain IPipelineInput<'b>.)

Each item taken is passed to the processor and router via pipeline (**|>**) and
Seq operations, recursively calling itself until an item can no longer be
retrieved from the buffer.

The implementation of IPipelineInput<'a>.Insert is the counterpart to the
previous function. It first tries to inset the item into the bounded blocking
queue, if this cannot be done then the overflow function is called if one is
present. Next the async consumer loop is started if it is not already running.
The idea behind this is that by keeping the payload processing running on the
thread pool while there is work to do it will cut down on the number of
context switches between threads.  Once an item cannot be taken from the
bounding blocking queue the loop will exit.

The rest of the code is pretty standard stuff and should be pretty easy to
follow.

I also define some symbolic operations to simply constructing and using the
pipeline:

**++>** Attaches the pipeline stage on the right hand side to the one on the left. **-->** Detaches the pipelinestage on the right from the one on the left. **<<--** Inserts a payload on the right into the pipeline stage on the left. **-->>** Inserts a payload on the left hand side into the pipeline stage on the right.  
These help to keep a nice terse description of the pipeline, once things get a little more complex other operators may be required, the now discontinued
[Axiom](http://msdn.microsoft.com/en-us/devlabs/dd795202.aspx) had a whole host of these, its a pity Microsoft dropped the language.

### Example

Heres a quick sample pipeline showing the pipeline in use:

* Stage 1 takes a string and splits it based on the ','.
* Stage 2 reverses each string.
* Stage 3 reverses the string back to the original.

```fsharp
module program
  open System
  open Pipeline  
  let consoleLock = new obj()  
  let split del n (s:string) =
    lock consoleLock (fun() ->
    do printfn "%A:before split %A" n s
    let split = s.Split([|del|])
    do printfn "%A:after: split into: %A" n split
    split |> Array.toSeq)  
  let reverse (s:string) =
    new string(s |> Seq.toArray |> Array.rev)  
  let oneToSingleton a b f=
    lock consoleLock (fun() ->
      printfn "%A:before reverse %A" a b
      let result = b |> f
      printfn "%A:after reverse %A" a result
      result|> Seq.singleton)  
  let OneToSeqRev a b = oneToSingleton a b reverse   
  ///Simply picks the first route
  let basicRouter( r, i) =
    let head = Seq.head r
    Seq.singleton head  
  let p1 = PipelineStage( split ',' "1", basicRouter)
  let p2 = PipelineStage( OneToSeqRev "2", basicRouter)
  let p3 = PipelineStage( OneToSeqRev "3", basicRouter)  
  p1 ++> p2 ++> p3 |> ignore  
  let generateCircularSeq (lst:'a list) =
    let rec next () =
      seq {
        for element in lst do
          yield element
        yield! next()
      }
    next()  
  for str in ["John,Paul,George,Ringo"]
  |> generateCircularSeq
  |> Seq.take 10
    do  str -->> p1  
  let x = Console.ReadKey()
```

As you can see the assignment of the pipeline stages is pretty simple as is the composition of multiple stages.  This was often one of the most difficult
areas while developing a similar pipelines in C# you could often find yourself with a few hundred lines of setup code which was a often a nightmare to debug
a few weeks later.

Hopefully I have whet your appetite with pipelines, in a future article I will be combining socket operations with pipeline stages to produce a flexible
framework to deal with high throughput network applications.

As always I appreciate any comments, until next time...
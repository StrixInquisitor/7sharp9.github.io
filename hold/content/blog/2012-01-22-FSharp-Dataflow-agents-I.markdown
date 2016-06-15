---
layout: post
author: 7sharp9
title: "F# Dataflow Agents Part I"
slug: FSharp-Dataflow-agents-I
date: 2012-01-22
comments: true
categories: [Programming, OldStuff]
tags: [Async Workflows, asynchronous, F#, Agents, TPL Dataflow, concurrency]
---
This is going to be a new series on using TPL Dataflow with F#.  First a little bit of history and background.

##TPL Dataflows heritage and background
TPL Dataflow or [(TDF)](http://msdn.microsoft.com/en-us/devlabs/gg585582) has been around for quite a 
while, it first surfaced more than a year ago as the successor to the Concurrency and Coordination Runtime 
[(CCR)](http://msdn.microsoft.com/en-us/library/bb648752.aspx) and with coming release of .Net 4.5 it will 
be part of the ```System.Threading.Tasks.Dataflow``` namespace.  Elements of the now halted project
[Axum](http://msdn.microsoft.com/en-us/devlabs/dd795202) are also present within the design of TDF.  

###Concurrency and Coordination Runtime (CCR)
CCR is a library that deals with asynchrony, concurrency, and coordination between blocks of asynchronous
code so that the programmer doesn't have to.  All of the low level details of synchronization and error 
propagation are taken care of in a consistent fashion.  CCR is still is included in 
[Microsoft Robotics Studio](ttp://www.microsoft.com/robotics/) where it is used extensively to exploit 
parallel hardware and deal with partial failure of systems. 

###Axum
Axum was another interesting Microsoft research project, it also utilized the actor model embracing the 
principles of isolation, and message-passing.  There was also extensive use  symbolic operators as a terse 
short hand way to indicate operations between actors.  For example ```<--``` defined a way to pass a message 
to an actor. Theres was also a similarity to CCR as Axum used the concepts of Ports and channels in a similar 
way.  It was a very interesting project and it was a shame it was put on hold.

###TPL Dataflow (TDF)
TDF builds on CCR and Axum, consolidating and refine to produce a more friendly fluent interface, much in 
the same vain as Language-Integrated Query [(LINQ)](http://msdn.microsoft.com/en-us/library/bb308959.aspx) 
and Reactive Extensions [(RX)](http://msdn.microsoft.com/en-us/data/gg577609).  

TDF is built around a number of different blocks which can be combined or linked together.  There are three 
different categories of blocks are as follows:

####Buffering Blocks
Buffering blocks simply buffer data in various ways before passing the data on to another block.  

   * ```BufferBlock<'T>```  - The BufferBlock act as a first-in-first-out (FIFO) queue, buffering each input.
   * ```BroadcastBlock<'T>``` - The BroadcastBlock linking to multiple targets copying the data to each of the 
   connected blocks.
   * ```WriteOnceBlock<'T>``` - The WriteOnceBlock acts like an immutable target, after an item first item 
   is passed to it, it effectively becomes read only.

####Executor Blocks
The executor blocks run user supplied code in the form of a lambda expressions or a ```Task<'T>```.

   * ```ActionBlock<'TInput>``` - The ActionBlock acts like the ```Action<'T>``` delegate performing an action 
   on each datum posted to it.
   * ```TransformBlock<'TInput,'TOutput>``` - The TransformBlock acts just like the ActionBlock except 
   that the action performed can have an output, this output is buffered and behaves just like a BufferBlock.
   * ```TransformManyBlock<'TInput,'TOutput>``` - The TransformManyBlock is just like a TransformBlock except 
   that is can produce more than one output for a given datum.

#### Joining Blocks
The Joining Blocks Combining or join data together in different ways.  

   * ```BatchBlock<'T>``` - The BatchBlock Combines multiple single items together, the items are 
   represented by arrays of elements.  The items are grouped together is batches and then passed on to 
   another block.
   * ```JoinBlock<'T1,'T2,…>``` - The JoinBlock acts as a form of ```Enumerable.Zip<'T1,'T2,'TResult>``` 
   except the zip operation is performed on the items in the source array.
   * ```BatchedJoinBlock<'T1,'T2,…>``` This block as the name suggests simply aggregates the JoinBlock and 
   the BatchBlock together.  

Thats an ultra high level tour thats only just scratches the surface.  I recommend you check out the 
[Introduction to TPL Dataflow](http://www.microsoft.com/download/en/details.aspx?id=14782) document to read 
up on the details.  Theres a few more resources in the [DevLabs area](http://msdn.microsoft.com/en-us/devlabs/gg585582) 
that you might find useful.  Hopefully this series should also shed a bit more light on TDF as we go along...

###F# Asynchronous Workflows and Agents
So where does that leave us in F#?  
In F# we have [Asynchronous Workflows](http://msdn.microsoft.com/en-us/library/dd233250.aspx) and 
[agents](http://msdn.microsoft.com/en-us/library/ee370357.aspx) and they help immensely in the concurrency 
and message passing, but that doest mean that we cant take advantage of the new features and refinements 
much in the same way as we can use Asynchronous Workflows to take advantage of Tasks.

This post is going to be centered around F# agents but with a twist.  First of all are going to be 
reimplementing a MailboxProcessor using TDF for the underlying processing.  This will allow us to to use 
all of our existing agent code and examples and also stay within the F# agent paradigm.  Following this 
approach we could also make use of the ```DataflowBlockOptions``` type, it has some interesting 
properties which we will look at in future posts:

   * TaskScheduler
   * CancellationToken
   * MaxMessagesPerTask
   * BoundedCapacity

###Implementation
In this post we are going replicate the MailboxProcessor, we will be using Tomas Petricek's caching agent 
example from [FSSnip](http://fssnip.net/8V)).  I have made a couple of modification to Tomas's code.  
I replaced the Dictionary type with a ConcurrentDictionary so that the caching agent could be called multiple 
times successively without the dictionary throwing an exception due to it already containing a key from a 
previous cached result.  I also changed the example code so that it requests cached HTML from the caching 
agent ten times with a 400ms interval in between each.  

{% codeblock Caching Agent Implementation lang:fsharp %}
module TplAgents
open System
open System.Collections.Generic
open System.Collections.Concurrent
open FsDataflow
open System.Net
open Microsoft.FSharp.Control.WebExtensions

type CachingMessage =
| Add of string * string
| Get of string * AsyncReplyChannel<option<string>>
| Clear

let caching = DataflowAgent.Start(fun agent -> async {
   let table = ConcurrentDictionary<string, string>()
   while true do
      let! msg = agent.Receive()
      match msg with
      | Add(url, html) -> 
         // Add downloaded page to the cache
         table.AddOrUpdate(url, html, fun k v -> html) |> ignore
      | Get(url, repl) -> 
         // Get a page from the cache - returns 
         // None if the value isn't in the cache
         if table.ContainsKey(url) then
            repl.Reply(Some table.[url])
         else
            repl.Reply(None) 
      | Clear -> 
           table.Clear() })
{% endcodeblock %}

{% codeblock Caching Agent Example lang:fsharp %}

/// Prints information about the specified web site using cache
let printInfo url = async {
   // Try to get the cached HTML from the caching agent
   let! htmlOpt = caching.PostAndAsyncReply(fun ch -> Get(url, ch))
   match htmlOpt with
   | None ->
       // New url - download it and add it to the cache
       use wc = new WebClient()
       let! text = wc.AsyncDownloadString(Uri(url))
       caching.Post(Add(url, text))   
       Console.WriteLine( sprintf "Download: %s (%d)" url text.Length)
   | Some html ->
       // The url was downloaded earlier 
       Console.WriteLine( sprintf "Cached: %s (%d)" url html.Length) }

let printfuncpro = printInfo "http://functional-programming.net"
// Print information about a web site -
// Run this repeatedly to use cached value
for i in 1 .. 10 do
   printfuncpro |> Async.Start
   Async.RunSynchronously <| Async.Sleep 400

// Clear the cache - 'printInfo' will need to
// download data from the web site again
Console.WriteLine(sprintf "Clearing the cache")
caching.Post(Clear)
printfuncpro |> Async.Start

Console.ReadKey() |> ignore
{% endcodeblock %}

Looking at the implementation above you can see that we need to implement the following members:

   * Start:```unit -> unit```
   * Receive:```?int -> Async<'Msg>```
   * Post:```'Msg -> unit```
   * PostAndTryAsyncReply:```(AsyncReplyChannel<'Reply> -> 'Msg) * ?int -> Async<'Reply option>```
   * PostAndAsyncReply:```(AsyncReplyChannel<'Reply> -> 'Msg) * int option -> Async<'Reply>```
   * static member Start:```(MailboxProcessor<'Msg> -> Async<unit>) * ?CancellationToken -> MailboxProcessor<'Msg>```

These are the only members we need to complete the caching agent example, I didn't want bamboozle everyone 
with an explosion of code from the onset so the remaining members will be implemented as and when we need 
them.  When we have implemented all the members from MailboxProcessor Ill post the full source on my
 [GitHub](https://github.com/7sharp9) account.

 The following members will be outstanding but it should be fairly trivial to implement them 
 once we have completed the code here.

   * PostAndReply:```(AsyncReplyChannel<'Reply> -> 'Msg) * int option -> 'Reply```
   * Scan:```('Msg -> Async<'T> option) * ?int -> Async<'T>```
   * TryPostAndReply:```(AsyncReplyChannel<'Reply> -> 'Msg) * ?int -> 'Reply option```
   * TryReceive:```?int -> Async<'Msg option>```
   * TryScan:```('Msg -> Async<'T> option) * ?int -> Async<'T option>```
   * CurrentQueueLength:```int```
   * DefaultTimeout:```int with get, set```

So here we go, this is the Dataflow implementation of the MailboxProcessor:
{% codeblock Dataflow Agent lang:fsharp %}
module FsDataflow
open System
open System.Threading
open System.Threading.Tasks
open System.Threading.Tasks.Dataflow
open System.Collections.Concurrent

type DataflowAgent<'Msg>(initial, ?cancellationToken) =
    let cancellationToken = 
        defaultArg cancellationToken Async.DefaultCancellationToken
    let mutable started = false
    let errorEvent = new Event<System.Exception>()
    let incomingMessages = new BufferBlock<'Msg>()
    let mutable defaultTimeout = Timeout.Infinite
    
    [<CLIEvent>]
    member this.Error = errorEvent.Publish

    member this.Start() =
        if started 
            then raise (new InvalidOperationException("Already Started."))
        else
            started <- true
            let comp = async { try do! initial this 
                               with error -> errorEvent.Trigger error }
            Async.Start(computation = comp, cancellationToken = cancellationToken)

    member this.Receive(?timeout) = 
        Async.AwaitTask <| incomingMessages.ReceiveAsync()

    member this.Post(item) = 
        let posted = incomingMessages.Post(item)
        if not posted then
            raise (InvalidOperationException("Incoming message buffer full."))

    member this.PostAndTryAsyncReply(replyChannelMsg, ?timeout) = 
        let timeout = defaultArg timeout defaultTimeout
        let resultCell = AsyncResultCell<_>()
        let msg = replyChannelMsg(AsyncReplyChannel<_>(fun reply -> resultCell.RegisterResult(reply)))
        let posted = incomingMessages.Post(msg)
        if posted then
            match timeout with
            |   Threading.Timeout.Infinite -> 
                    async { let! result = resultCell.AsyncWaitResult
                            return Some(result) }  
            |   _ ->
                    async { let! ok =  resultCell.GetWaitHandle(timeout)
                            let res = (if ok then Some(resultCell.GrabResult()) else None)
                            return res }
        else async{return None}

    member this.PostAndAsyncReply( replyChannelMsg, ?timeout) =                 
            let timeout = defaultArg timeout defaultTimeout
            match timeout with
            |   Threading.Timeout.Infinite -> 
                let resCell = AsyncResultCell<_>()
                let msg = replyChannelMsg (AsyncReplyChannel<_>(fun reply -> resCell.RegisterResult(reply) ))
                let posted = incomingMessages.Post(msg)
                if posted then
                    resCell.AsyncWaitResult  
                else
                    raise (InvalidOperationException("Incoming message buffer full."))
            |   _ ->            
                let asyncReply = this.PostAndTryAsyncReply(replyChannelMsg, timeout=timeout) 
                async { let! res = asyncReply
                        match res with 
                        | None ->  return! raise (TimeoutException("PostAndAsyncReply TimedOut"))
                        | Some res -> return res }           
                    
    static member Start(initial, ?cancellationToken) =
        let dfa = DataflowAgent<'Msg>(initial, ?cancellationToken = cancellationToken)
        dfa.Start()
        dfa
{% endcodeblock %}

The crux of the implementation from TDF's point of view is the use of the BufferBlock.  
This is one of the most fundamental blocks within TDF.  Its the equivalent of the ```Port<'T>``` 
type from CCR and the ```Mailbox``` type from F# which is used internally within the 
MailboxProcessor.  As mentioned abouve the BufferBlock type is a first-in-first-out (FIFO) buffer 
and is responsible for buffering any data that is Posted to it.  

OK, I'm going to leave it at that for now while you digest the code presented here.  

In part II I will be drilling into the detail on whats going on internally and also describing more 
of the TDF model, so tune in soon for Part II.  

Until next time...
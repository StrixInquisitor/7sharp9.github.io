---
layout: post
title: "FSharp Dataflow agents III"
date: 2012-02-20
slug: FSharp-Dataflow-agents-III
comments: true
categories: [programming]
series: [dataflowagents]
tags: [Async Workflows, asynchronous, F#, Agents, TPL Dataflow, concurrency]
description: ""
type: post
---
This will be the last post on rebuilding the `MailboxProcessor` using [TDF](http://msdn.microsoft.com/en-us/devlabs/gg585582), 
here's a quick discussion of the missing pieces...

First, lets start with the simple ones, these don't really require much discussion.

### DefaultTimeout
```fsharp
let mutable defaultTimeout = Timeout.Infinite

member x.DefaultTimeout
   with get() = defaultTimeout
   and set(value) = defaultTimeout <- value
```
This simply provides a mutable property using `Timeout.Infinite` as a default setting.

### CurrentQueueLength
```fsharp
member x.CurrentQueueLength() = incomingMessages.Count 
```
Another simple one, this methods uses into the underlying `BufferBlock` to extract its current queue length using its `Count` property.

### TryReceive
```fsharp
member x.TryReceive(?timeout) = 
    let ts = TimeSpan.FromMilliseconds(float <| defaultArg time out defaultTimeout)
    Async.AwaitTask <| incomingMessages.ReceiveAsync(ts)
                           .ContinueWith(fun (tt:Task<_>) -> 
                                             if tt.IsCanceled || tt.IsFaulted then None
                                             else Some tt.Result)
```
Here we get a little help from [TPL](http://msdn.microsoft.com/en-us/library/dd460717.aspx) to apply a continuation on completion 
using `ContinueWith`.  We use a lambda to return either `None`, in a time out condition, or `Some tt.Result` when we successfully receive an item.  

### TryPostAndReply
```fsharp
type AsyncResultCell<'a>() = 
    ...
	member x.TryWaitResultSynchronously(timeout:int) = 
	    //early completion check
	    if source.Task.IsCompleted then 
	        Some source.Task.Result
	    //now force a wait for the task to complete
	    else 
	        if source.Task.Wait(timeout) then 
	            Some source.Task.Result
	        else None

member x.TryPostAndReply(replyChannelMsg, ?timeout) :'Reply option = 
    let timeout = defaultArg timeout defaultTimeout
    let resultCell = AsyncResultCell<_>()
    let msg = replyChannelMsg(new AsyncReplyChannel<_>(fun reply -> resultCell.RegisterResult(reply)))
    if incomingMessages.Post(msg) then
        resultCell.TryWaitResultSynchronously(timeout)
    else None
```
Things get a little more interesting from here on in.  Firstly we need to add a new synchronisation member to the `AsyncResultCell<'a>` type: `TryWaitResultSynchronously`.   We again enlist the help of the TPL primitives to check for the early completion using `source.Task.IsCompleted` returning the result if it is there, otherwise we use the `Task` property's `Wait` method to check the item returns within the time out interval.  In the usual manner, `Some source.Task.Result` is returned or `None` for a failure.  

### PostAndReply
```fsharp
member x.PostAndReply(replyChannelMsg, ?timeout) : 'Reply = 
    match x.TryPostAndReply(replyChannelMsg, ?timeout = timeout) with
    | None ->  raise (TimeoutException("PostAndReply timed out"))
    | Some result -> result
```
This one wraps a call to `TryPostAndReply` with some pattern matching.  In the event of a time out `None` is returned from `TryPostAndReply` in this instance we raise a `TimeoutException` otherwise we unwrap the result from the option using `| Some result -> result`.

### TryScan
```fsharp
member x.TryScan((scanner: 'Msg -> Async<_> option), timeout): Async<_ option> = 
    let ts = TimeSpan.FromMilliseconds( float timeout)
    let rec loopForMsg = async {
        let! msg = Async.AwaitTask <| incomingMessages.ReceiveAsync(ts)
                                      .ContinueWith(fun (tt:Task<_>) -> 
                                          if tt.IsCanceled || tt.IsFaulted then None
                                          else Some tt.Result)
        match msg with
        | Some m ->  let res = scanner m
                     match res with
                     | None -> return! loopForMsg
                     | Some res -> return! res 
        | None -> return None}                             
    loopForMsg
```
This one also uses the same `ContinueWith` functionality in the recursive `loopForMsg` function, perhaps some 
of these functions could extracted out and refactored but I prefer to keep the code like this to better explain what's going 
on.  The the code is available on GitHub anyway so feel free to clean up any detritus and send me a pull request.  Again we use pattern matching to keep calling the `loopForMsg` function until the result is returned or a time out occurs.  

### Scan
```fsharp
member x.Scan(scanner, timeout) =
    async { let! res = x.TryScan(scanner, timeout)
            match res with
            | None -> return raise(TimeoutException("Scan TimedOut"))
            | Some res -> return res }
```
Finally we have Scan, this is much like PostAndReply in that it just acts as a wrapper around `TryScan` making use of 
pattern matching throwing an exception on a time out.

That sums up the last few pieces, completing the TDF implementation of the `MailboxProcessor`.  I think this series of posts has shown the elegance of F#'s asynchronous workflows.  The use of recursive functions and the compositional nature of asynchronous workflows really helps when you are doing this type of programming.  It's also very nice on the eye, each section being clearly defined.

The more astute of you may have noticed something a little different.  `Scan` and `TryScan` are destructive in this implementation, the unmatched messages are purged from the internal queue.  Although I could have mirrored the same functionality of the `MailboxProcessor` by using an internal list to keep track of unmatched messages, this leads to performing checks during `Receive` and `Scan` and their derivatives to make sure that this list is used first when switching from `Scan` and `Receive` functionality.  

I think the separation of concerns are a little fuzzy in the `MailboxProcessor`.  The `scan` function seems like an after thought, even if you don't use `Scan` you still pay a price for it as there are numerous checks between the internal queue and the unmatched messages list.  You can also run into issues while using `Scan` and `TryScan` that can result in out of memory conditions due to the inherent unbounded nature.  I will briefly describe and explore the conditions that can lead to that in the next post.  In the implementation presented here we can get bounded checking by passing in an optional `DataflowBlockOptions` and setting a value for the `BoundedCapacity` property.  

**EDIT:** The code for this series of articles is now available on GitHub: [FSharpDataflow](https://github.com/7sharp9/FSharpDataflow)

Until next time...

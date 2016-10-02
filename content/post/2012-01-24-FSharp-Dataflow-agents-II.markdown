---
layout: post
author: 7sharp9
title: F# Dataflow Agents Part II
slug: FSharp-Dataflow-agents-II
date: 2012-01-30
comments: true
categories: [programming]
series: [dataflowagents]
tags: [Async Workflows, asynchronous, FSharp, Agents, TPL Dataflow, concurrency]
description: ""
type: post
---
Right, no messing about this time, straight to the code. 
## Construction
This is pretty straight forward and I don't want to detract from the important bits of this post, the only thing 
of note is the `cancellationToken` which is initialized to a default value using the `defaultArg` function if the 
optional parameter  `cancellationToken` is not supplied. The TDF construct that we to use to perform most of the hard 
work is `incomingMessages` which is a `BufferBlock<'Msg>`.
```fsharp
type DataflowAgent<'Msg>(initial, ?cancellationToken) =
    let cancellationToken = 
        defaultArg cancellationToken Async.DefaultCancellationToken
    let mutable started = false
    let errorEvent = new Event<System.Exception>()
    let incomingMessages = new BufferBlock<'Msg>()
    let mutable defaultTimeout = Timeout.Infinite
```

## Error
This is the public facing part for the Error event.  The `[<CLIEvent>]` attribute exposes the event in a friendly manner to other .Net languages by adding the `add_Error` and `remove_Error` event handler properties to allow subscription to take place.  The `Error` event fires when an exception is thrown in the `initial` asynchronous workflow.  
```fsharp
[<CLIEvent>]
member this.Error = errorEvent.Publish
```

## Start
This is implemented the same as the MailboxProcessor.  An exception is thrown if the agent has already started as this is not valid operation.  We set the mutable field `started` to true and proceed to start the `initial` asynchronous workflow.  This workflow is wrapped in a `try with block` so that if an exception is thrown we catch it and trigger the `Error` event.  The computation is then started with `Async.Start(...)`.
```fsharp
member this.Start() =
    if started 
        then raise (new InvalidOperationException("Already Started."))
    else
        started <- true
        let comp = async { try do! initial this 
                           with error -> errorEvent.Trigger error }
        Async.Start(computation = comp, cancellationToken = cancellationToken)
```

## Receive
The Receive member is used by the agent as a way of waiting for a message to arrive without blocking.  Because the TDF functionality is all TPL Task based we use the the [Async](http://msdn.microsoft.com/en-us/library/ee370232.aspx) helper functions.  In this instance we utilise the Async.AwaitTask passing in the `incomingMessages` `ReceiveAsync` method to wait for 
a message to arrive.  The integration between F# async and TDF is nice and succinct here.
```fsharp
member this.Receive(?timeout) = 
    Async.AwaitTask <| incomingMessages.ReceiveAsync()
```

## Post
The Post member allows a message to be sent to the agents, this member simply calls the `incomingMessages` `Post` method passing in the `item`.  We raise an exception if there is a problem posting (i.e. the `incomingMessages` internal queue is full). 
```fsharp
member this.Post(item) = 
    let posted = incomingMessages.Post(item)
    if not posted then
        raise (InvalidOperationException("Incoming message buffer full."))
```

## PostAndTryAsyncReply / PostAndAsyncReply
I'm grouping both of these together as they are related in functionality.  In the previous post I purposely left 
out some ancillary code as it added unnecessary complexity to the introduction.  There are a two types we need to be able to replicate the `PostAndTryAsyncReply` and `PostAndAsyncReply` members of the `MailboxProcessor`.

### AsyncReplyChannel
The first type we need is the `AsyncReplyChannel<'Reply>`.  This type takes a function that accepts a generic `'Reply` and returns a unit.  It is used as a way of communicating back to the caller of the `PostAndTryAsyncReply` and `PostAndAsyncReply` members via its single member `Reply`.  This should become a little clearer when we see it used in context.  

An `AsyncRepyChannel` does actually exist in F# under the Microsoft.FSharp.Control namespace and is used my the `MailboxPRocessor`, unfortunately its constructor is marked as internal so we are not able to reuse it here.

```fsharp
type AsyncReplyChannel<'Reply>(replyf : 'Reply -> unit) =
    member x.Reply(reply) = replyf(reply)
```

### AsyncResultCell
The next type we need is the `AsyncResultCell<'a>`.  We use this as a way to await for the results of an asynchronous operation.  We create a [TaskCompletionSource](http://msdn.microsoft.com/en-us/library/dd449174.aspx) (`source`), which is a TPL type that we use as a way of signalling to a callback / lambda expression when a message has arrived.  

**RegisterResult** is used as a way of notifying when a message has been arrived, this is used internally by our agent as a result of a reply being made to the AsyncReplyChannel.

**AsyncWaitResult** is a continuation wrapper, it is called when we want to wait indefinitely for the result to be returned.  It wraps a successful completion with a call to task.Result which then returns the result.

**GetWaitHandle** is used as a mechanism to force the asynchronous result to return within a specified timeout interval.  If a result is not returned within the timeout then this function will return false.

**GrabResult** returns the result from the TaskCompletionSource object `source`.  This is set earlier by the `RegisterResult` member.
```fsharp
type AsyncResultCell<'a>() =
    let source = new TaskCompletionSource<'a>()
    member x.RegisterResult result = source.SetResult(result)

    member x.AsyncWaitResult =
        Async.FromContinuations(fun (cont,_,_) -> 
            let apply = fun (task:Task<_>) -> cont (task.Result)
            source.Task.ContinueWith(apply) |> ignore)

    member x.GetWaitHandle(timeout:int) =
        async { let waithandle = source.Task.Wait(timeout)
                return waithandle }
                
    member x.GrabResult() = source.Task.Result
```

#### PostAndTryAsyncReply
This one is a little more tricky and I have added a few line number references to try and make it easier.  On **line 3** we 
declare an resultCell to collect the result of the asynchronous operation.  This is used on **line 4** when we create a `msg` 
to post to `incomingMessages` on **line 5**.  The `replyChannelMsg` is a function that takes an `AsyncReplyChannel` and returns 
a message, so we create an `AsyncReplyChannel` with a lambda expression that registers the reply with the `resultCell`.  This 
is the key to how this works, you have to remember that will be done the other side of the operation which will be within the 
asynchronous processing loop of the agent when `Reply` is called on the `AsyncReplyChannel`.  

Finally pattern matching is used on **line 7** to call either `AsyncWaitResult` or `GetWaitHandle` on the `resultCell`.  The `AsyncWaitResult` function is used to wait indefinitely and the `GetWaitHandle` function is used if we want to use a timeout.  Both of these are asynchronous workflows that either return a result or return an option type containing the result.  

```fsharp
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
```

#### PostAndAsyncReply
This member uses the same functionality as `PostAndTryAsyncReply`, creating a message using the `AsyncReplyChannel`.  The main 
difference is that an asynchronous workflow is created that wraps a call to `PostAndTryAsyncReply` if the `timeout` is specified.

```fsharp
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
```

## Static Start
The static Start function is used as a way to construct and start the agent than using the constructor and then calling the Start function.  This is really just a simple short cut for this common use case.
```fsharp
static member Start(initial, ?cancellationToken) =
    let dfa = DataflowAgent<'Msg>(initial, ?cancellationToken = cancellationToken)
    dfa.Start()
    dfa
```

Until next time...
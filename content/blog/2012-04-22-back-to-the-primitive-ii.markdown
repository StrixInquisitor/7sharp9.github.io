---
author: 7sharp9
layout: post
title: "Back to the Primitive II"
slug: Back-To-The-Primitive-II
date: 2012-04-22
comments: true
categories: [programming]
series: [backtotheprimitive]
tags: [F#, C#, TPL, Async, Threading]
description: ""
type: post
---
In the last post I discussed an asynchronous version of the `ManualResetEvent` and as promised this time we will be looking at an
 asynchronous version of the `AutoResetEvent`.  I'm using [Stephen Toubs post](http://blogs.msdn.com/b/pfxteam/archive/2012/02/11/10266923.aspx) 
as reference and we will be building a version that is functional in style that maps straight into asynchronous work flows without and conversion 
or adaptors.  

### What is an AutoResetEvent?
An `AutoResetEvent` can be described as a turnstile mechanism, it lets a single waiting person through before re-latching 
waiting for the next signal.  This is opposed to a `ManualResetEvent` which functions like an ordinary gate. Calling Set opens 
the gate, allowing any number of threads that are waiting to be let through. Calling Reset closes the gate.  

### AsyncAutoResetEvent
First of all here is the shape of the type that we will be building:

```
type AsyncAutoResetEvent =
    new : ?reusethread:bool -> AsyncAutoResetEvent
    member Set : unit -> unit
    member WaitAsync : unit -> Async<bool>
```

Fairly simple: implied constructor, `Set` and `WaitAsync` members.  
### Implied Constructor
Thinking about this logically we may need the following items:

*   A queue mechanism to store asynchronous waiters - `let mutable awaits = Queue<_>()`.
*   A way of knowing if a signal has been made in the absence of any waiters - `let mutable signalled = false`.
*   We can also declare a short-circuit asynchronous workflow for the situation that `Set()` is called before `WaitAsync()` 
- `let completed = async.Return true`.  This will save us constructing an `AsyncResultCell<_>` and going though the 
rest of the asynchronous mechanism.  

Also notice that an optional parameter called `reusethread` is defined, we use the `?` prefix when defining it to make it 
optional.  We then make use of the `defaultArg` function to give it a default value of false if a one is not passed in.  This 
will be used in the `Set` operation to determine if the code will run on the same thread or a thread in the ThreadPool.  
```
open System
open System.Threading
open System.Collections.Generic
 
    type AsyncAutoResetEvent(?reusethread) =
		let mutable awaits = Queue<_>()
		let mutable signalled = false
        let completed = async.Return true
        let reuseThread = defaultArg reusethread false
```
	
### WaitAsync()

The first step is to use  a locking construct to control access to the mutable queue `awaits`.  Inside this lock we 
check to see if `signalled` is true and if so we reset it to false and return our pre-built `completed` asynchronous workflow.  If 
signalled is false then we create a new `AsyncResultCell<_>` and add it to the queue then return the `AsyncResult` to the caller.  

```
        member x.WaitAsync() =
            lock awaits (fun () ->
                if signalled then
                    signalled <- false
                    completed
                else
                    let are = AsyncResultCell<_>()
                    awaits.Enqueue are
                    are.AsyncResult)
```

### Set()

We first declare a function called `getWaiter()`, we use this function to return an [option type](http://msdn.microsoft.com/en-us/library/dd233245.aspx)
 that is either `Some AsyncResultCell<bool>` or `None`.  We use the lock function to control access to the mutable queue `lock awaits`.  Once 
inside the lock we use pattern matching to capture `awaits.Count` and `signalled`:     

*   The first pattern match `(x,_)` checks if there are any waiters (`awaits.Count > 0`) and then dequeues an `AsyncResultCell<bool>` from the 
	queue and returns it within an option type: `Some <| awaits.Dequeue()`.  
*   The second pattern match `(_,y)` checks whether `signalled` is set to false before setting its value to true.  This causes next `WaitAsync()` 
	caller to get the short-circuited value `completed`.  This means that an `AsyncResultCell<bool>` does not need to be created and go though the 
	whole async mechanism.  We then return `None` as there is no waiter to be notified.  
*   The final pattern match `(_,_)` is used when there are no waiting callers and `signalled` has already being set, there is simply nothing to do in 
	this situation so we return `None`.  

We use the `getWaiter()` function via pattern match.  If we have a result i.e. Some AsyncResultCell<bool> then we call `RegisterResult` 
passing in `AsyncOK(true)` to indicate a completion.  Notice that we also pass in the `reuseThread` boolean that was declared as part of the 
constructor.  If `reuseThread` is true then the notification to the waiter happens **synchronously** use this with care!  Personally I would stick 
with the default of false to ensure that the operation is completed via the thread pool, unless you have a performance critical reason and the 
waiting code that executes is **very fast**.  

```
		member x.Set() =
		    let getWaiter()=
		        lock awaits (fun () ->
		            match (awaits.Count, signalled) with
		            | (x,_) when x > 0 -> Some <| awaits.Dequeue()
		            | (_,y) when not y -> signalled <- true;None
		            | (_,_) -> None)
		    match getWaiter() with
		    | Some a -> a.RegisterResult(AsyncOk(true), reuseThread)
		    | None _ -> ()
```

The reason for using the `getWaiter()` function is to separate the locking function away from the notification, if `RegisterResult` 
was called within the lock and `reuseThread` was true then the awaiting function would be called synchronously within the lock which 
would not be a very good situation to be in.  


So there we have it, I could take this series further and convert the other primitives that Stephen Toub describes but there should be 
enough information in these two posts to set you on your way.  If anyone would like me to complete the series then let me know.  I 
may well finish them off and post them on GitHub in the future, time permitting.

Thanks for tuning in, until next time...
* * *
#### Musical inspiration during the creation of this post:  
*   Pantera - Cowboys From Hell  
*   Cacophony - Go Off  

  {{< img-fit
    2u "http://upload.wikimedia.org/wikipedia/en/a/a8/CowboysFromHell.jpg"  "Pantera - Cowboys From Hell"
    2u "http://upload.wikimedia.org/wikipedia/en/0/09/Cacophony_-_1988_-_Go_Off%21.jpg" "Cacophony - Go Off" "" "" >}}

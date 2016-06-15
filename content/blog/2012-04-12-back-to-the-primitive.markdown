---
author: 7sharp9
layout: post
title: "Back to the Primitive"
slug: Back-To-The-Primitive
date: 2012-04-12
comments: true
categories: [programming]
series: [backtotheprimitive]
tags: [F#, Async, TPL, Threading]
description: ""
type: post
---
In this post we are going **back to the primitive**.  No it's not about the same named song by Soulfly, *(which incidentally does contains F# notes)* but a return to thread synchronisation primitives and their asynchronous counterparts.

We are going to be looking at an asynchronous version of the [ManualResetEvent](http://msdn.microsoft.com/en-us/library/system.threading.manualresetevent.aspx).  This was 
recently covered by Stephen Toub on the [pfx team blog](http://blogs.msdn.com/b/pfxteam/archive/2012/02/11/10266920.aspx).  We will be taking a slightly different view on 
this as we will be using asynchronous workflows which will give us nice idiomatic usage within F#.    

First lets look of the shape of the type that Stephen defined:

```
public class AsyncManualResetEvent 
{ 
    public Task WaitAsync(); 
    public void Set(); 
    public void Reset(); 
}
```
Now this can be used from within F# by using the `Async.AwaitTask` function from the Async module but this is like wrapping one asynchronous paradigm with another, and 
although this does work, what if you want to avoid the overhead of wrappers and stay strictly within async workflows.

```fsharp
type asyncManualResetEvent() =
    member x.WaitAsync() : unit -> Async<bool>
    member x.Set() : unit -> unit
    member x.Reset() : unit -> unit
```
That's what we want to see!  I don't want to get into the details of the description of how the C# version works as Stephen does a very good job of that already.  What I will explain though is how we essentially do the same thing while staying with the realm of functional programming.  As we are getting into the lower lever details no doubt we will have to start relying on some low level locking primitives like Monitors, Semaphores, and Interlocked operations, even the F# core libraries have a 
cornucopia of those.  

Lets look at the first member `WaitAsync()`.  The first step is to create a something to store the result of the operation, all we will just be storing and returning 
asynchronously is a boolean to indicate that the wait handle has been set.  To do this we use one of the types from the 
[F# power pack](http://fsharppowerpack.codeplex.com/) `AsyncResultCell<'T>`.  I think that such a type should of been exposed from the F# core libraries but it was 
omitted for some reason.  There is a type called `ResultCell<'T>` with much the same functionality in the FSharp.Core.Control namespace but it is marked internal so 
it's not available for our use.  

We declare a [reference cell](http://msdn.microsoft.com/en-us/library/dd233186.aspx) of type `AsyncResultCell<'T>` and then create the `WaitAsync()` member, all we have 
to do is dereference the value of the reference cell with `!` and call its `AsyncResult` member, this gives us an `Async<bool>` which we can easily use in an asynchronous workflow.

```fsharp
type asyncManualResetEvent() =
    let aResCell = ref <| AsyncResultCell<_>()

    member x.WaitAsync() = (!aResCell).AsyncResult
```
The next bit is fairly simple too.  All we need to do is dereference the value of the reference cell, and invoke the `RegisterResult` member by passing in a value of
 `AsyncOk(true)`.  The boolean value of true will be used by the type inference system to constrain the value of the `Async<_>` returned from `WaitAsync`.

```fsharp
    member x.Set() = (!aResCell).RegisterResult(AsyncOk(true))
```
The last part is the most complex *(as usual)*.  Here we create a recursive function called `swap` that will try to exchange the `AsyncResultCell<'T>` for a new 
one.  We dereference the reference cell to `currentValue`, then we use a CAS (Compare And Swap) operation to compare the `aResCell` with `currentValue` and if they 
are equal `newVal` will replace `aResCell`.  On the next line if the result of the CAS operation means that `result` and `currentValue` are equal then we are finished, 
otherwise we spin the current thread for 20 cycles using `Thread.SpinWait 20` before retrying the operation via recursion `swap newVal`.  This will be a lot less 
expensive than switching to user or kernel mode locking, and the period of contention between threads should be very small.  Finally the swap operation is started 
by passing in a new `AsyncResultCell<'T>`.    

There are various other methods we could of used, for instance we could of wrapped a `ManualResetEvent` with a call to `Async.AwaitWaitHandle`, although this 
would of meant using the kernel mode locking of the `ManualResetEvent` which is a bit more expensive.

In Stephen Toub's post he mentions Task's being orphaned due to the `Reset()` method being called before the `Task<'T>` has been completed, that shouldn't happen in our 
implementation due the the closures being stored internally for completion by the async infrastructure.  Heres a quick test harness to make sure everything works as expected anyway.

```fsharp
    member x.Reset() =
        let rec swap newVal = 
            let currentValue = !aResCell
            let result = Interlocked.CompareExchange<_>(aResCell, newVal, currentValue)
            if obj.ReferenceEquals(result, currentValue) then ()
            else Thread.SpinWait 20
                 swap newVal
        swap <| AsyncResultCell<_>()
```
```fsharp
let amre = asyncManualResetEvent()
let x = async{let! x = amre.WaitAsync()
              Console.WriteLine("First signalled")}

let y = async{let! x = amre.WaitAsync()
             Console.WriteLine("Second signalled")}

let z = async{let! x = amre.WaitAsync()
              Console.WriteLine("Third signalled")}
//start async workflows x and y
Async.Start x
Async.Start y

//reset the asyncManualResetEvent, this will test whether the async workflows x and y 
// are orphaned due to the AsyncResultCell being recycled.
amre.Reset()

//now start the async z
Async.Start z

//we set a single time, this should result in the three async workflows completing
amre.Set()

Console.ReadLine() |> ignore
```
Here we can see everything works out as we expected:

{{< figure src="https://lh6.googleusercontent.com/-NYIKC5Gaahs/T4YAQGtP9RI/AAAAAAAABR8/_cTOriC1_Fw/amre.png" >}}

Thats all there is too it, next time I will be exploring an asyncAutoResetEvent in much the same vein.  

Until next time...
* * *
#### Musical inspiration during the creation of this post:  
*   Smashing Pumpkins - Zeitgeist  
*   Soulfly - Primitive
*   FooFighters - FooFighters  
    
{{< img-fit 
  2u "http://upload.wikimedia.org/wikipedia/en/f/fb/Zeitgeist_cover.png" "Smashing Pumpkins Zeitgeist" 
  2u "http://upload.wikimedia.org/wikipedia/en/3/34/Primitive.png" "Soulfly Primitive"
  2u "http://upload.wikimedia.org/wikipedia/en/0/0d/FooFighters-FooFighters.jpg" "FooFighters" "" "" "" >}}

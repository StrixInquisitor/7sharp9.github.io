---
author: 7sharp9
date: '2011-07-03'
layout: post
slug: agent-based-scheduling
status: publish
title: Agent based scheduling
wordpress_id: '382'
comments: true
categories:
- Programming
- OldStuff
tags:
- Agents
- Async Workflows
- distributed systems
- F#
---

One of the areas that I am very interested in is agents and I have been doing
quite a lot of work in this area lately.

Agents can be used for a multitude of different purposes ranging from:
isolated message passing, object caching, finite state machines, web crawling,
and even reactive user interfaces.  One of the ideas that I have been looking
into lately is agent based scheduling.<!-- more -->

##SchedulerAgent

****Listing 1**** shows a simple Agent based scheduler:

{% codeblock listing 1 lang:fsharp %}
    module AgentUtilities
    open System
    open System.Threading 
 
    //Agent alias for MailboxProcessor
    type Agent<'T> = MailboxProcessor<'T> 
 
    /// Two types of Schedule messages that can be sent
    type ScheduleMessage<'a> =
      | Schedule of ('a -> unit) * 'a * TimeSpan * TimeSpan * CancellationTokenSource AsyncReplyChannel
      | ScheduleOnce of ('a -> unit) * 'a * TimeSpan * CancellationTokenSource AsyncReplyChannel
  
    /// An Agent based scheduler
    type SchedulerAgent<'a>()=   
      let scheduleOnce delay msg receiver (cts: CancellationTokenSource)=
        async { do! Async.Sleep(delay)
            if (cts.IsCancellationRequested)
            then cts.Dispose()
            else msg |> receiver }  
      let scheduleMany initialDelay  msg receiver delayBetween cts=
        let rec loop time (cts: CancellationTokenSource) =
           async { do! Async.Sleep(time)
               if (cts.IsCancellationRequested)
               then cts.Dispose()
               else msg |> receiver
               return! loop delayBetween cts}
        loop initialDelay cts  
      let scheduler = Agent.Start(fun inbox ->
        let rec loop() = async {
          let! msg = inbox.Receive()
          let cs = new CancellationTokenSource()
          match msg with
          | Schedule(receiver, msg:'a, initialDelay, delayBetween, replyChan) ->
            Async.StartImmediate(scheduleMany
                         (int initialDelay.TotalMilliseconds)
                         msg
                         receiver
                         (int delayBetween.TotalMilliseconds)
                         cs )
            replyChan.Reply(cs)
            return! loop()
          | ScheduleOnce(receiver, msg:'a, delay, replyChan) ->
            Async.StartImmediate(scheduleOnce
                         (int delay.TotalMilliseconds)
                         msg
                         receiver
                         cs)
            replyChan.Reply(cs)
            return! loop()
        }
        loop())  
      ///Schedules a message to be sent to the receiver after the initialDelay.
      ///  If delaybetween is specified then the message is sent reoccuringly at the delaybetween interval.
      member this.Schedule(receiver, msg, initialDelay, ?delayBetween) =
        let buildMessage replyChan =
          match delayBetween with
          | Some(x) -> Schedule(receiver,msg,initialDelay, x, replyChan)
          | _ -> ScheduleOnce(receiver,msg,initialDelay, replyChan)
        scheduler.PostAndReply (fun replyChan -> replyChan |> buildMessage)
{% endcodeblock %}
The structure of the SchedulerAgent broken down into sections below:

### ScheduleMessage

Lines **9-11** show the definition of ScheduleMessage.  This is a discriminated
union of two different types of Schedule message.

#### ScheduleOnce

ScheduleOnce has four parameters: 

  1. A function which is called at the schedule time ('a -> unit).
  2. The message that is sent at the schedules time ('a).
  3. A TimeSpan which is the length of time to wait before triggering the schedule.
  4. An AsyncReplyChannel<CancellationTokenSource>(CancellationTokenSource AsyncReplyChannel).  This is used to return a CancellationTokenSource which can be used to cancel the Schedule.

#### Schedule

Schedule has five parameters which are as follows:

  1. A function which is called at the schedule time ('a -> unit).
  2. The message that is sent at the schedules time ('a).
  3. A TimeSpan which is the initial length of time to wait before first triggering the schedule function.
  4. A TimeSpan which is used as an interval between each subsequent triggering of the schedule function.
  5. An AsyncReplyChannel<CancellationTokenSource>(CancellationTokenSource AsyncReplyChannel).  This is used to return a CancellationTokenSource which can be used to cancel the Schedule.

## SchedulerAgent

### scheduleOnce

Lines **16-20** define an async workflow, which asynchronously sleeps for the specified time before checking that the schedule hasn't been cancelled before finally calling the schedule function. 

### scheduleMany

Lines **22-29** define a recursive async workflow, which asynchronously sleeps for the specified interval (_3rd Parameter_) before checking the schedule hasn't been cancelled before finally calling the schedule function. The **loop** function is then called passing in the second TimeSpan interval _(4th Parameter)_. 

### scheduler

This is the main processing loop for the agent.  A recursive **loop** function
is declared on line **32**.  On line **33** the agent waits for a message
to arrive.  Once a message arrives a **CancellationTokenSource** is created on
line **36** which can be used to cancel an already scheduled message.
Pattern matching is used on line **35** to find the type of message that has
been received.  The first pattern matching block on lines **36-43** matches
the **Schedule** message.  The parameters from the Schedule message are passed
into the **scheduleMany** function.  This is then invoked asynchronously via
the **Async.StartImmediate** function.  The CancellationTokenSource is now
returned to the caller on line **43**. This allows the caller to cancel an
already running schedule.   Finally the recursive **loop** function is called
on line **44**.  The second pattern matching block on lines **45-52** is much
the same passing the parameters from the **ScheduleOnce** message into the
**scheduleOnce** function, again this is invoked via the
**Async.StartImmediate** function.  Like the Schedule message the
CancellationTokenSource returned on line **51** and the recursive **loop**
function is called on line **52**.

The agent is then started on line **51** by calling the **loop** function for the first time.

### Members

The SchedulerAgent has only a single member **Schedule**.  This member
function takes three parameters and an optional parameter **delayBetween**.  A
function called **buildMessage** on line **59** uses the optional parameter
with pattern matching to determine whether a **ScheduleOnce** or a
**Schedule** message is created.  The agent is posted the correct message type
on line **63** using the synchronous call scheduler.PostAndReply.  We use a
synchronous call to return the cancellationTokenSource immediately, and this
can be used to cancel a running schedule.

  
##Sample Application

**Listing 2** shows a test harness that creates and uses a simple string based message scheduler.

{% codeblock Listing 2 lang:fsharp %}
    open AgentUtilities
    open System
    open System.Threading  
    let scheduler = SchedulerAgent<_>()
    let printer message =
      printfn "%s: %s" (DateTime.Now.TimeOfDay.ToString()) message  
    let singlecancel = scheduler.Schedule(printer,
                        "Hello from the scheduler",
                        TimeSpan(0,0,0,5))  
    let multicancel = scheduler.Schedule( printer,
                        "Hello from the multi scheduler",
                        TimeSpan(0,0,0,5),
                        TimeSpan(0,0,0,0,500))  
    printfn "Press any key to cancel."
    Console.ReadKey() |> ignore  
    //Cancel the multi scheduler
    multicancel.Cancel()
    printfn "Cancelled, press any key to exit."
    Console.ReadKey() |> ignore
{% endcodeblock %}

I hope this gives you a feel for what you can do with agent based scheduling.
The library here could be expanded further in several ways.  You could replace
the fixed message with a message generator function or even an agent based
message generator.  If the schedule function was abstracted somewhat it could
be made to accept an agent as the receiver.

One of the key areas I am looking at is building a distributed agent library
that would allow an agent to communicate over network layers transparently.  A
scheduler agent would be even more powerful in this environment.  I could
envisage them used for a many different things in this environment:  heart
beat messages, performance sampling, diagnostics and testing.

Until next time...
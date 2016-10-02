---
author: 7sharp9
date: '2011-02-01'
layout: post
slug: pipeline-processing-1
status: publish
title: Pipeline processing 1
wordpress_id: '237'
comments: true
categories:
- programming
series: [pipelineprocessing]
tags:
- asynchronous
- Complexity
- FSharp
- Pipeline
- TPL
description: ""
type: post
---

### Welcome to new series of articles on pipeline processing.

First up, what's a pipeline?  Well according to [Wikipedia](http://en.wikipedia.org/wiki/Pipeline_(computing)):

>A pipeline is a set of data processing elements connected in series, so that
the output of one element is the input of the next one. The elements of a
pipeline are often executed in parallel or in time-sliced fashion; in that
case, some amount of buffer storage is often inserted between elements.  

In essence its a way of dealing with complexity and its also a way of breaking
down a process into separate tasks of a similar size.  If they are used
correctly then pipelines can be used to increase the overall throughput of a
system.<!-- more -->

In enterprise systems or in fact in most large systems, a simple idea or
program can rapidly become overwhelmingly complex.  The management all of the
disparate parts of the system can become a nightmare and the code can quickly
becomes a labyrinth, navigating it becomes a skill of only the most
accomplished code _ninja, _and even then your playing Russian roulette with
any bug fixes.In an effort to keep things manageable and simple one approach
that we can use is a pipeline. The idea is that each stage is connected to one
or more other stages and each that each stage deals with a single task before
passing the work onto the next stage.  There are many primitive types in the
[Task Parallel Library](http://msdn.microsoft.com/en-us/library/dd460717.aspx)
(TPL) that you could use to compose a working pipeline, we will be using a
lightweight subset taking only a few core ideas and making sure we get a nice
slick design that is both powerful and flexible.

Here's a quick flow diagram of the sort of thing that we will be looking at:

{{< figure src="https://lh5.googleusercontent.com/-55hM6Bez26w/TwTtHHXc-ZI/AAAAAAAABPo/LOgX1UywK0I/pipeline-tuv.png" >}}

This is a generic asynchronous payload based pipeline.  Each stage is
asynchronous and self contained and is connected to one or more other stages.
As a payload enters the pipeline it is initially added to a bounding blocking
queue.  If the queue is full then the payload is said to have overflowed and
is passed to the failure processor where the payload can be processed or
transformed in some way before being passed to a failure router which would in
turn pass the payload to one or more of the next failure stages.  The same is
also true for a successfully queued payload except that the payload is first
dequeued, processed, then passed to a router which then passes the payload to
one or more stages.  If an exception occurs during processing then the payload
is passed to the failure processor and processed like an overflow.  I am
purposely missing out any details of asynchronous operation as they will be
described in more detail next time.

We will be using a little bit of [Language Oriented Programming](http://tomasp.net/blog/fsharp-iv-lang.aspx) 
to construct the pipeline stages, maybe using a little bit of operator overloading too.  
I will describe all of this in more detail next time as we dig into the code.  
I want this to be just a brief introduction to what we are going to be doing.

Here's a more detailed description of the components that are involved in each
stage:

## Bounded Blocking Queue

This is a standard bounded blocking queue from the TPL, its purpose here is to
limit the amount of payloads that are waiting to be processed, each queue will
have an associated time-out period, if the time-out period passes the payload
is passed to the failure processor for processing and then finally to the the
failure router to be passed to one or more failure stages.

## Processors

Each pipeline processor has a primary Processor<T,U> and a failure processor<T,V>.

The primary processors job is to convert type T to type U, both types can be
the same if you wish, you may well be thinking why would I want a processing
stage that essentially leaves the type unchanged?  In this case the processor
acts as a simple a pass through but using this you to do some custom routing.
This can be very be useful in some scenarios and I will describing this in
more detail in a further post.

Each pipeline stage also has a failure processor<T,V>.  The failure processor
acts on the payload to produce the desired type and passes it onto the failure
router.  The reasoning behind this scheme rather than a simplistic exception
logger is simply flexibility.  Having spent a lot of time with this kind of
API in a more locked down format I have found that you can end up wanting a
bit more flexibility especially when some developers try to get a bit creative
with the API or start state to the payload.  A good example of having some
flexibility is during overflow:  If the bounded blocking queue fills up and
blocks for the time-out period then the payload could be passed to a failure
failure processor in which types T and V are the same.  This would allow us to
pass the payload to another stage and retry later on by attaching some sort of
delayed forwarding pipeline stage.

## Routers

The router is responsible for getting the payload to the next pipeline stage,
it can be implemented as a simple predicate function operating on the type
directly or even some outside influence if you wish.   An example of this
might be a simple duplicating stage where the payload is passed to multiple
output stages rather than just one, or a time based router where one stage is
passed the payload during the day and another at night.   When you start to
think about the possibilities the Processor / Router combination can be really
really flexible.

Each pipeline stage also has a corresponding has a failure router, this can be
used for all sorts of purposes like routing the failed payload to a logging
component, routing to a delayed retry mechanism, or saved to a database etc.

Thats all for now, we will be digging into some code and more detail next
time, and I will be describing a few different types of pipelines so you can
get a feel of how to use them and the overall structure.

Another interesting aspect of these pipelines is that once constructed they
can be composed into single reusable blocks that as a whole, represent a
single pipeline stage.  These composite stages can then be connected together
to form a super pipeline stage, complexity is only visible when you start to
drill down and becomes almost fractal like...

As always please leave any comments or suggestions.
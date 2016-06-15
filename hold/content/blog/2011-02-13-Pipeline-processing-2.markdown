---
author: 7sharp9
date: '2011-02-13'
layout: post
slug: pipeline-processing-2
status: publish
title: Pipeline processing 2
wordpress_id: '260'
comments: true
categories:
- Programming
- OldStuff
tags:
- asynchronous
- Complexity
- F#
- parallelism
- Performance
- Pipeline
---

### Welcome to pipeline processing part 2.

I feel I need to backtrack slightly from the previous post, having worked with
pipelines for quite some time I have the advantage of knowing all of the
details that may be alluded to in these articles without being effected by any
omissions I may make, obviously you guys aren't in that position, so I'm going
to try and rectify that a bit now.  If you have any queries then please leave
a comment and I will try to address them in further articles. Pipelines are a
simple concept but in practice there can be some caveats and things to bear in
mind, sometime the whole mindset of development team can be against them
unless they can see the bigger picture...<!-- more -->

First of all one of the most important things to bear in mind with a pipeline
is that you are only going to be as fast as your slowest stage, if one stage
is ten times slower than another then it will be waiting for input most of the
time, we need to make this more efficient.

#### Premature Optimisation

Lots of developers out there have the [premature optimisation is the root of
all evil](http://www.c2.com/cgi/wiki?PrematureOptimization) mindset and will
quote this out loud to you when you mention performance early on.  I'm not
advocating premature optimisation, in this instance performance is key, if one
stage is out of kilter with the rest then we are going to be running at that
pace of the slowest stage, if that's too slow for the requirements then you
are screwed.

The more I think about performance the more I believe its an essential part of
creating code. There are too many developers these days that will produce
sloppy unrefined plain bad code.  I'm a keen believer in producing quality
code that you can be proud of, and part of that is having clean code that's
both efficient and works.  I think some of this boils down to a feature driven
approach that measures developers solely in terms of features added, take the
typical [burn down chart](http://en.wikipedia.org/wiki/Burn_down_chart) that
you would use in [agile software development](http://en.wikipedia.org/wiki/Agile_software_development):

![](http://alistair.cockburn.us/get/1880)

There is nowhere on this chart that measures whether the code is good or bad or runs to performance requirements.  In the future I may do an article on
integrating code quality into your build process, its something I have been thinking about doing for a while now.

While I'm talking about performance you also might want to check out Joe Duffy's post on [The 'premature optimization is evil' myth](http://www.bluebyt
esoftware.com/blog/2010/09/06/ThePrematureOptimizationIsEvilMyth.aspx), and also check out Joe's book on [concurrent programming](http://www.bluebytesoftw
are.com/books/winconc/winconc_book_resources.html), put it on your wish list if you haven't already read it, its a great book.

#### Unbalanced pipelines

Data is received from the network via packets, each packet may contain one or more messages from a business systems or indeed a partial message.  We need to
collect the packets either separate or combine them to form individual messages, deserialize them and finally log them.

Here's a sample pipeline demonstrating an unbalanced pipeline:

{% img https://lh6.googleusercontent.com/-HDFpPk4zBzY/TwTnGvEn9kI/AAAAAAAABPE/CcRYtQ4fsEQ/pipeline.png %}

  1. Stage 1 of the pipeline receives these packets and processes them into individual messages passing them onto Stage 2.
  2. We now have a complete message (in this instance the message will be XML) we want to turn it into a .Net type we now deserialize the message and pass it onto Stage 3.
  3. To keep this pipeline simple all we are going to do here is log type of message to disk or a database, the pipeline is now complete.  

Stage 1 would take 5 seconds to fully utilise stage 2, stage 2 would take 2
seconds to fully utilise stage 3.  You can see this pipeline will only process
100 transactions per second even though stages 2 has 5x the throughput of
stage 1 and stage 3 has 2x the throughput of stage 2.  Our efficiency is only
about 10% of what it could be, we must be able to do something about that.

Lets look at the following diagram which demonstrate a balanced pipeline:

#### Balanced pipelines

{% img https://lh4.googleusercontent.com/-8Pgq9ISPe4Q/TwTp1nwBzfI/AAAAAAAABPU/AUwLor1WI7o/balanced-pipeline.png %}

You can see from this diagram that each stage processes the same number of
transactions per second by introducing parallel stages.  This is called a
balanced pipeline.  Sometimes you cant get a perfectly balanced pipeline but
you should strive to get as close as possible.  Sometimes a certain stage
cannot be parallelised because it may have mutable state, or you are using
some sort of [IOC](http://en.wikipedia.org/wiki/Inversion_of_control)
container for processing services, this might make constructing the various
stages in parallel difficult, this can become an art form in itself and can
lead to very large initialisation sections in the code.  I hope to address all
of these issues in due course.

This poses some interesting thoughts and questions to add to some you may
already have:

  * How can we easily manage the complexity of parallelism?
  * How will the distribution of work be handled?
  * How do you baseline the throughput of each stage?
  * Can you automate the parallelism of a particular stage?
  * How do you manage the complexity of multiple stages?
  * What about parallelism and mutable state?  

The final point to note is the Distributor/Router must operate at a much
higher rate than the processing stages otherwise you will introduce another
bottle neck into the system, although you could have a multiple distributors
but this would yet another degree of complexity that has to be managed.  You
can see that things can quickly become more complicated than they first
seemed.

I know I promised lots of funky code but I figured there was a bit more
explaining to do before we can get to that.  I want to take a more of an
iterative approach to show you the potential pitfalls that can occur during
developing such a pipeline and how to avoid them.  I thought this would be a
lot more constructive than dropping a load of code and some pretty pictures
and hoping for the best.

Next time we will be exploring a simple pipeline stage with a single degree of
parallelism and a simple router.  After that we will then start exploring and
answering the questions above, adding more features like parallelism,
instrumentation, and visualisation.

Hope you enjoyed this even though there was no code!

See you next time.


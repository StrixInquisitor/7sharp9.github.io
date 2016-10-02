---
author: 7sharp9
date: '2011-01-13'
slug: sockets-and-bockets-1
status: publish
title: Sockets and Bockets 1
wordpress_id: '39'
comments: true
series: [socketsandbockets]
tags:
- Async Workflows
- CSharp
- FSharp
- GC
- Memory fragmentation
- Sockets
categories:
- programming
description: ""
---

### Welcome to part 1

A while back I read an interesting article by _Brian McNamara_ [f-async-on-the-server-side](http://lorgonblog.wordpress.com/2010/03/28/f-async-on-the-server-side/) 
which describes C# and F# versions of a simple asynchronous
socket server, one of the driving forces behind the article was how F# can
wrap the traditional asynchronous model with [Asynchronous Workflows](http://msdn.microsoft.com/en-us/library/dd233250.aspx), this
produces nice clean simple code compared to the C# version which uses lambda
expressions, the code looks quite ugly in this style!  However thats not the
end of the story, a lot of memory fragmentation can occur using the APM model
when there is a high throughput, so I thought I would see if I could take this
a step further...<!-- more -->

There are some lesser known methods that were added to the [Socket](http://msdn.microsoft.com/en-us/library/system.net.sockets.socket.aspx) 
class in .Net 2.0 SP1: ReceiveAsync, SendAsync, ConnectAsync and DisconnectAsync.
These methods use an event driven model and **do not** result in the creation
of AsyncResult objects, these are created on every asynchronous call in the
traditional Socket Begin/End methods.  Once you have thousands of clients
sending and receiving thousands of messages all of the object creation can
really have an adverse effect on performance on the garbage collected, you
will regularly see the AsyncResult objects hitting Generation 1 and 2.

To use the xxxAsync methods you have pass a [SocketAsyncEventArgs](http://msdn.microsoft.com/en-us/library/system.net.sockets.socketasynceventargs.aspx)object which is
assigned callback method and a buffer, the callback method called
asynchronously when the operation completes and is passed the corresponding
SocketAsyncEventArgs object, this allows you query the buffer in a receive
operation.

The scope of this series of articles is to initially replicate Brian's demo
using F# and a pool of SocketAsyncEventArgs and a contiguous block of memory
to hold the data being sent and received on the Socket, this again further
reduces memory fragmentation on the send and receive buffers.

I have successfully developed an enterprise server for a client using this
method, it processed thousands of simultaneous connected clients and messages,
key components in the system were the High performance sockets, a pipeline
processor and a highly efficiency means of data compaction, I will only be
including the High performance sockets in this series but the other components
will be at a later date in separate articles.  Interestingly all of the code
was originally developed in c# but had a distinctly functional style, even the
Pipeline Processing is reminiscent of functional composition using the F#
pipeline operator **|>** although an analogue of attach and detach was used
which in itself is declarative.

Although there is no code in this article there is plenty in the next!

Please feel free to leave comments or add any suggestions, hope to see you
next time...
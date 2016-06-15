---
author: 7sharp9
date: '2011-01-20'
layout: post
slug: sockets-and-bockets-part-3
status: publish
title: Sockets and Bockets 3
wordpress_id: '139'
comments: true
categories:
- Programming
- OldStuff
tags:
- Async Workflows
- F#
- Memory fragmentation
- Performance
- SAEA
- Sockets
---

## Welcome to part three!

As promised heres a description of the inner workings.  I'm sick to death of
typing SocketAsyncEventArgs so from now on I will refer to it as SAEA.<!-- more -->

**BocketPool**  
The BocketPool has an interesting name and with it an interesting constructor!
It takes the following parameters:

**number**: The number of items to create in the BocketPool. **size**: The size of each buffer in bytes. **callback**: A callback function which is invoked whenever the SAEA object completes its operation.

{% codeblock lang:fsharp %}
type BocketPool( number, size, callback) as this =
    let number = number
    let size = size
    let totalsize = (number * size)
    let buffer = Array.create totalsize 0uy
    let pool = new BlockingCollection<SocketAsyncEventArgs>(number:int)
{% endcodeblock %}

A **buffer** is created with a size equal to the (**number** * **size**)
in bytes.

{% codeblock lang:fsharp %}
do
      let rec loop n =
        match n with
        | x when x < totalsize ->
          let saea = new SocketAsyncEventArgs()
          saea.Completed |> Observable.add callback
          saea.SetBuffer(buffer, n, size)
          this.CheckIn(saea)
          loop (n + size)
        | _ -> ()
      loop 0
{% endcodeblock %}

The tail recursive loop function creates a SAEA object and adds it to the
BlockingCollection(pool).

The buffer is assigned to each SAEA but each is given a unique offset to use,
this is done by the SetBuffer method.  Using this method of allocation, memory
fragmentation is reduced to a minimum by allowing the same buffer to be
reused.

We use the pipeline operator to attach the **Completed** event to the callback
method that is passed in the constructor.

The CheckIn, CheckOut, and Count methods are simply wrappers around the
BlockingCollection.

We also implement **IDisposable** to take care of the disposal of the SAEA in the
BlockingCollection.

**Connection**  
The main purpose for this type is to encapsulate the sending and receiving of
messages for a particular client. A BocketPool is created for both the send
and receive operations, the receiveCompleted and SentCompleted are invoked
when the respective operations complete.

**Send**

{% codeblock lang:fsharp %}
member this.Send (msg:byte[]) =
  let s = sendPool.CheckOut()
  Buffer.BlockCopy(msg, 0, s.Buffer, s.Offset, msg.Length)
  socket.SendAsyncSafe(this.sendCompleted, s)
{% endcodeblock %}

Initially a Bocket is checked out of the sendPool using sendPool.Checkout(),
the **msg** byte array is copied to the corresponding **Offset** property of
the SAEA.

Finally the SendAsyncSafe extension method is called passing in the SAEA and
the callback.

**sendCompleted**

{% codeblock lang:fsharp %}
member this.sendCompleted (args: SocketAsyncEventArgs) =
      try
        match args.LastOperation with
        | SocketAsyncOperation.Send ->
          match args.SocketError with
          | SocketError.Success ->
            ()
          | SocketError.NoBufferSpaceAvailable
          | SocketError.IOPending
          | SocketError.WouldBlock ->
            if not(anyErrors) then
              anyErrors <- true
              failwith "Buffer overflow or send buffer timeout"
          | _ -> args.SocketError.ToString() |> printfn "socket error on send: %s"
        | _ -> failwith "invalid operation, should be receive"
      finally
        sendPool.CheckIn(args)
{% endcodeblock %}

This function matches the LastOperation property of the SAEA using pattern
matching, this ensures that the LastOperation is always SocketError.Success.

We raise exceptions on NoBufferSpaceAvailable, IOPending, and WouldBlock as
buffer overflows and match any other conditions the wildcard.

Finally we Check the Bocket back in so that it can be reused.

**receiveCompleted**

{% codeblock lang:fsharp %}
member this.receiveCompleted (args: SocketAsyncEventArgs) =
      try
        match args.LastOperation with
        | SocketAsyncOperation.Receive ->
          match args.SocketError with
          | SocketError.Success ->
            socket.ReceiveAsyncSafe( this.receiveCompleted, receivePool.CheckOut())
            let data = Array.create args.BytesTransferred 0uy
            Buffer.BlockCopy(args.Buffer, args.Offset, data, 0, data.Length)
            let client = args.RemoteEndPoint
            args.RemoteEndPoint <- null
            data |> printfn "received data: %A"
          | _ -> args.SocketError.ToString() |> printfn "socket error on receive: %s"
        | _ -> failwith "unknown operation, should be receive"
      finally
        receivePool.CheckIn(args)
{% endcodeblock %}

This function is very similar to the sendCompleted and could probably be
refactored a bit using the [Hole in the middle
pattern](http://enfranchisedmind.com/blog/posts/the-hole-in-the-middle-
pattern/).  Again we check to ensure the last operation was a success, we
checkout another Bocket and start another ReceiveAsyncSafe. This ensures that
the socket can begin another receive operation as soon as possible while we
take the data from the SAEA Buffer, we do this with Buffer.Block copy.

If this were a fully-fledged API then we would raise an event here so that
users of the component could consume the data.

In my own component the data is inserted into a series of processing stages
using the [Pipeline Pattern](http://www.cise.ufl.edu/research/ParallelPatterns
/PatternLanguage/AlgorithmStructure/Pipeline.htm), which I will be may
describe in a future post if anyone's interested.

**TcpListener**  
The TcpListener is very similar to the Connection object in that it has a pool
of SAEA objects that are used to accept connection from clients, again a round
of refactoring could be done here to avoid duplication with the Connection
type.  The main difference is that we don't need to use the Buffer on the SAEA
to send anything to the client when it initially connects.

**acceptCompleted**

{% codeblock lang:fsharp %}
member this.acceptcompleted (args : SocketAsyncEventArgs) =
      try
        match args.LastOperation with
        | SocketAsyncOperation.Accept ->
          match args.SocketError with
          | SocketError.Success ->
            listeningSocket.AcceptAsyncSafe( this.acceptcompleted, acceptPool.Take())
            //create new connection
            let connection = newConnection args.AcceptSocket
            connection.Start()  
            //update stats
            reportConnections   
            //async start of messages to client
            startSending connection  
            //remove the AcceptSocket because we will be reusing args
            args.AcceptSocket <- null
          | _ -> args.SocketError.ToString() |> printfn "socket error on accept: %s"
        | _ -> args.LastOperation |> failwith "Unknown operation, should be accept but was %a"
      finally
        acceptPool.Add(args)
{% endcodeblock %}
	
This function is similar to the send and receive completed methods in the
Connection type, although this time we create a Connection object and call the
Start function, this puts the Connection into receive mode.

The reportConnections is called next which simply prints how many clients are
connected, we now start an Asyncronous workflow using the startSending
function.

Finally we set the AcceptSocket property to null on the SAEA object and add it
back to the BlockingCollection so that it can be reused.

The purpose of the BlockingCollection here is to have a fixed pool of SAEA
that block when there isn't an SAEA to service the new connection, this could
be a potential issue for the client as it could timeout while waiting for a
connection but this is a far preferable situation than causing your server to
be effectively denied service due to overload.

**startSending**

{% codeblock lang:fsharp %}
let startSending connection =
      Async.Start (async {
        try
          use _holder = connection
          do! asyncServiceClient connection
        with e ->
          if not(anyErrors) then
            anyErrors <- true
            Console.WriteLine("server ERROR")
          raise e
        } )
{% endcodeblock %}

This function uses the syntactic sugar of the asynchronous workflows to start
an operation on the Thread pool, once queued on the thread pool it is wrapped
in a using block with the **_use _holder = connection_** statement and
asynchronously calls the asyncServiceClient function, this has the effect of
disposing of the Connection type when it exits scope or encounters an
exception.

**asyncServiceClient**

{% codeblock lang:fsharp %}
let asyncServiceClient (client: Connection) = async {
      client.Send(header)
      while true do
        do! asyncWriteStockQuote(client) }
{% endcodeblock %}

This function sends a one byte header message to the client using the
Connection.Send, followed by calling asyncWriteStockQuote in a continuous
loop.

**asyncWriteStockQuote**

{% codeblock lang:fsharp %}
let asyncWriteStockQuote(connection:Connection) = async {
      do! Async.Sleep 1000
      connection.Send(testMessage)
      Interlocked.Increment(&numWritten) |> ignore }
{% endcodeblock %}
	
This function sleeps for 1000ms and uses the Connection.Send function to sent
a message to the client, the number of results is updated using the
Interlocked class.

I would like to refer you to [Brian McNamara's
post](http://lorgonblog.wordpress.com/2010/03/28/f-async-on-the-server-side/)
that describes this part in more detail.  The only difference in our workflow
is that we don't use a stream operation as we have the SendAsyncSafe function
to do all the work for us.  IDispose is also implemented on this type too as
we have to dispose of the SAEA objects that are used for the asynchronous
accepts.

**createTcpSocket**

{% codeblock lang:fsharp %}
let createTcpSocket() =
      new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp)
{% endcodeblock %}

This function simply wraps the Sockets class constructor mapping it to: Tcp
protocol, Streaming, and InterNetwork Address type.

**createListener**

{% codeblock lang:fsharp %}
let createListener (ip:IPAddress, port, backlog) =
      let s = createTcpSocket()
      s.Bind(new IPEndPoint(ip, port))
      s.Listen(backlog); s
{% endcodeblock %}
This function calls the createTcpSocket function, binds to the IPAddress and
port that are passed in and starts listening for connections.

**Start**

{% codeblock lang:fsharp %}
member this.start () =
      listeningSocket.AcceptAsyncSafe( this.acceptcompleted, acceptPool.Take())
      while true do
      Thread.Sleep 1000
      let count = Interlocked.Exchange(&numWritten, 0)
      count |> printfn "Quotes per sec: %A"
{% endcodeblock %}

This function starts the whole process of listening for a connection from
clients.  A SAEA is taken from the BlockingCollection and AcceptAsyncSafe is
called.

I have tried to describe all of the functions that I think merit a description
but I have been involved in this sort of code for years now so if you have any
queries feel free to just drop a comment and I will try to help.

When looking through the code remember that this is just a demo, I am
currently still working on a few things but may offer the full API available
for download at a later date or put it on GitHub.

In part four we are going to compare some of the differences in operation
between the xxxAsync and the IAsync pattern, obviously there is a lot more
code and inherent complexity in this implementation but in high volume
situations it makes a lot of difference.

See you next time.  
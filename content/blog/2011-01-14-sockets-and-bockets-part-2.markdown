---
author: 7sharp9
date: '2011-01-14'
layout: post
slug: sockets-and-bockets-part-2
status: publish
title: Sockets and Bockets 2
wordpress_id: '68'
comments: true
categories:
- programming
series: [socketsandbockets]
tags:
- Async Workflows
- F#
- Memory fragmentation
- Performance
- Sockets
description: ""
type: post
---

### Welcome to part two

Lets jump in at the deep end and take a look at some code...

When you look at the method syntax for the xxxAsync methods you will notice
they return a boolean value that indicates if the method completed
synchronously, this means that you have to check the return value every time
you use one of the methods and invoke the callback yourself if it completes
synchronously.  In practice this hardly ever happens, and normally only on a
send operation.  But as it is a possibility we will add module with a some
extension methods in to help us out, this will make the code more readable and
avoid unnecessary duplication.<!-- more -->

### SocketExtensions

```fsharp
module SocketExtensions
  open System
  open System.Net
  open System.Net.Sockets  
  type Socket with
    /// extension method to make async based call easier, this ensures the callback always gets
    /// called even if there is an error or the async method completed syncronously
    member s.InvokeAsyncMethod( asyncmethod, callback, args:SocketAsyncEventArgs) =
      let result = asyncmethod args
      if result <> true then callback args
    member s.AcceptAsyncSafe(callback, args) = s.InvokeAsyncMethod(s.AcceptAsync, callback, args)
    member s.ReceiveAsyncSafe(callback, args) = s.InvokeAsyncMethod(s.ReceiveAsync, callback, args)
    member s.SendAsyncSafe(callback, args) = s.InvokeAsyncMethod(s.SendAsync, callback, args)
    member s.DisconnectAsyncSafe(callback, args) = s.InvokeAsyncMethod(s.DisconnectAsync, callback, args)
```

Now lets get down to business, the next few types have a fair bit of code in
them so I will briefly explain each type in turn:

### BocketPool

A BocketPool is a combination of a
[SocketAsyncEventArgs](http://msdn.microsoft.com/en-
us/library/system.net.sockets.socketasynceventargs.aspx) object and a chunk of
memory allocated in an array.  The array is sliced up into sections and
allocated for each send or receive operation by setting a start and end index
using [SetBuffer()](http://msdn.microsoft.com/en-us/library/bb549836.aspx).
If you remember last time I mentioned that a lot of memory fragmentation can
occur during sending and receiving due to continuously allocating memory
buffers on the Socket object, this is primarily done through the BeginSend and
BeginReceive methods passing in a byte array.  Using the BocketPool it a great
way of reducing the amount of garbage collection during heavy traffic.

The other major difference with SocketAsyncEventArgs is the way in which you
make the send and receive calls, heres the general flow that occurs:

* Create a SocketAsyncEventArgs object or get one from a pool.
* Allocate an array to the buffer.
* Allocate an offset and length to the buffer.
* Allocate a callback method.
* Call Socket.xxxAsync passing in the SocketAsyncEventArgs, the operation will complete and invoke the callback.  

What we are going to do is wrap the whole creation, array allocation, and
offsetting to the BocketPool:

```fsharp
namespace Fes
  open System
  open System.Net.Sockets
  open System.Collections.Concurrent  
  type BocketPool( number, size, callback) as this =
    let number = number
    let size = size
    let totalsize = (number * size)
    let buffer = Array.create totalsize 0uy
    let pool = new BlockingCollection<SocketAsyncEventArgs>(number:int)
    let mutable disposed = false
    let cleanUp() =
      if not disposed then
        disposed <- true
        pool.CompleteAdding()
        while pool.Count > 1 do
          (pool.Take() :> IDisposable).Dispose()
        pool.Dispose()
    do
      let rec loop n =
        match n with
        | x when x < totalsize ->
          let saea = new SocketAsyncEventArgs()
          saea.Completed |> Observable.add( fun saea -> (callback saea))
          saea.SetBuffer(buffer, n, size)
          this.CheckIn(saea)
          loop (n + size)
        | _ -> ()
      loop 0
    member this.CheckOut()=
      pool.Take()
    member this.CheckIn(saea)=
      pool.Add(saea)
    member this.Count =
      pool.Count
    interface IDisposable with
      member this.Dispose() = cleanUp()
```

Next up we have to look at the Connection and the Tcplistener types as two
interconnected entities:

* The TcpListener listens for a connection on a socket and port number.
* The client connects to the server.
* An accept socket is allocated to the client, at this point we have one socket for the server and once for each client.
* We also need to allocate a BocketPool for send and receive operation for each client
To simplify things we are going to encapsulate the accept socket management
into a type, it will also need a corresponding BocketPool to service any send
and receive operations to and from the client

### Connection

```fsharp
namespace Fes
  open System
  open System.Net
  open System.Net.Sockets
  open System.Collections.Generic
  open System.Collections.Concurrent
  open System.Threading
  open SocketExtensions  
  type Connection(maxreceives, maxsends, size, socket:Socket) as this =
    let socket = socket
    let maxreceives = maxreceives
    let maxsends = maxsends
    let sendPool = new BocketPool(maxsends, size, this.sendCompleted )
    let receivePool = new BocketPool(maxreceives, size, this.receiveCompleted)
    let mutable disposed = false
    let mutable anyErrors = false  
    let cleanUp() =
      if not disposed then
        disposed <- true
        socket.Shutdown(SocketShutdown.Both)
        socket.Disconnect(false)
        socket.Close()
        (sendPool :> IDisposable).Dispose()
        (receivePool :> IDisposable).Dispose()  
    member this.Start() =
      socket.ReceiveAsyncSafe(this.receiveCompleted, receivePool.CheckOut())  
    member this.Stop() =
      socket.Close(2)  
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
    member this.sendCompleted (args: SocketAsyncEventArgs) =
      try
        match args.LastOperation with
        | SocketAsyncOperation.Send ->
          match args.SocketError with
          | SocketError.Success -> ()
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
    member this.Send (msg:byte[]) =
      let s = sendPool.CheckOut()
      Buffer.BlockCopy(msg, 0, s.Buffer, s.Offset, msg.Length)
      socket.SendAsyncSafe(this.sendCompleted, s)
```

Finally here's the TcpListener type.  It is responsible for creating an
initial Connection object for each client and starts asynchronous sending
messages to that client once a second, also notice that there is another
BlockingCollection involved, this is somewhat simpler than the usage in the
bocketPool as we have no buffer to manage here.

* _It is possible to fill the initial Buffer property, this causes the buffer to be sent to the client as soon as it has connected to the server, this can be useful to sent initial data to the client, such as protocol definitions etc)_
A finite number of connections can occur before blocking will occur depending
on the number of AsyncEventArgs in the collection, this stops potential denial
of service attacks due to too many connection being made.

### TcpListener

```fsharp
namespace Fes
  open System
  open System.Net
  open System.Net.Sockets
  open System.Collections.Generic
  open System.Collections.Concurrent
  open System.Threading
  open SocketExtensions  
  type TcpListener(maxaccepts, maxsends, maxreceives, size, port, backlog) as this =  
    let createTcpSocket() =
      new Socket(AddressFamily.InterNetwork, SocketType.Stream, ProtocolType.Tcp)  
    let createListener (ip:IPAddress, port, backlog) =
      let s = createTcpSocket()
      s.Bind(new IPEndPoint(ip, port))
      s.Listen(backlog); s  
    let listeningSocket = createListener( IPAddress.Loopback, port, backlog)  
    let initPool (maxinpool, callback) =
      let pool = new BlockingCollection<SocketAsyncEventArgs>(maxinpool:int)
      let rec loop n =
        match n with
        | x when x < maxinpool ->
          let saea = new SocketAsyncEventArgs()
          saea.Completed |> Observable.add callback
          pool.Add saea
          loop (n+1)
        | _ -> ()
      loop 0
      pool  
    let acceptPool = initPool (maxaccepts, this.acceptcompleted)
    let newConnection socket = new Connection (maxreceives, maxsends, size, socket)
    let testMessage = Array.init<byte> 128 (fun _ -> 1uy)
    let header = Array.init<byte> 1 (fun _ -> 1uy)
    let mutable disposed = false  
    //mutable state from original
    let mutable anyErrors = false
    let mutable requestCount = 0
    let mutable numWritten = 0  
    //async code from original
    let asyncWriteStockQuote(connection:Connection) = async {
      do! Async.Sleep 1000
      connection.Send(testMessage)
      Interlocked.Increment(&numWritten) |> ignore }  
    //async code from original
    let asyncServiceClient (client: Connection) = async {
      client.Send(header)
      while true do
        do! asyncWriteStockQuote(client) }  
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
    let reportConnections =
      Interlocked.Increment(&requestCount) |> ignore
      if requestCount % 1000 = 0 then
        requestCount |> printfn "%A Clients accepted"  
    let cleanUp() =
      if not disposed then
        disposed <- true
        listeningSocket.Shutdown(SocketShutdown.Both)
        listeningSocket.Disconnect(false)
        listeningSocket.Close()  
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
    member this.start () =
      listeningSocket.AcceptAsyncSafe( this.acceptcompleted, acceptPool.Take())
      while true do
      Thread.Sleep 1000
      let count = Interlocked.Exchange(&numWritten, 0)
      count |> printfn "Quotes per sec: %A"  
    member this.Close() =
      cleanUp()  
    interface IDisposable with
      member this.Dispose() = cleanUp()
```

Its a fair bit of code to take in at once, so Ill leave you with it to ponder
over.  Ill be explaining all of the interesting bits in more detail in part
three...

Please feel free to leave any comments you have, especially on better use of
functional constructs.  
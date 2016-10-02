---
author: 7sharp9
date: '2011-06-05'
layout: post
slug: agents-and-objectpools
status: publish
title: ' Agents and ObjectPools'
wordpress_id: '341'
comments: true
categories:
- programming
tags:
- Agents
- Async Workflows
- asynchronous
- FSharp
- Performance
description: ""
type: post
---
Everyone knows F# agents are cool right?  Well here's yet another example of how versatile they can be...

There was a series of posts last April by Stephen Toub from the [pfxteam](http://blogs.msdn.com/b/pfxteam/) at Microsoft.  I was reading
through some of the posts again the other day and thought some of the ideas presented there would make interesting projects in F# to demonstrate the
flexibility and succinctness of the language.  I thought the [ObjectPool](http://blogs.msdn.com/b/pfxteam/archive/2010/04/13/9990427.aspx)
example would make an interesting project in F# using agents aka MailboxProcessors.  An ObjectPool is basically a pool of objects that have been
pre-created so that you can grab one and use it, and then place it back in the pool when you're finished.  They are useful in situations where the cost of
creating object from scratch is very high or you want to cut down on allocations in the garbage collector.<!-- more -->

First of all heres the C# code as it was presented in the Parallel Extensions download:

```
using System.Collections.Generic;
using System.Diagnostics;  
namespace System.Collections.Concurrent
{
  /// <summary>Provides a thread-safe object pool.</summary>
  /// <typeparam name="T">Specifies the type of the elements stored in the pool.</typeparam>
  [DebuggerDisplay("Count={Count}")]
  [DebuggerTypeProxy(typeof(IProducerConsumerCollection_DebugView<>))]
  public sealed class ObjectPool<T> : ProducerConsumerCollectionBase<T>
  {
    private readonly Func<T> _generator;  
    /// <summary>Initializes an instance of the ObjectPool class.</summary>
    /// <param name="generator">The function used to create items when no items exist in the pool.</param>
    public ObjectPool(Func<T> generator) : this(generator, new ConcurrentQueue<T>()) { }  
    /// <summary>Initializes an instance of the ObjectPool class.</summary>
    /// <param name="generator">The function used to create items when no items exist in the pool.</param>
    /// <param name="collection">The collection used to store the elements of the pool.</param>
    public ObjectPool(Func<T> generator, IProducerConsumerCollection<T> collection)
      : base(collection)
    {
      if (generator == null) throw new ArgumentNullException("generator");
      _generator = generator;
    }  
    /// <summary>Adds the provided item into the pool.</summary>
    /// <param name="item">The item to be added.</param>
    public void PutObject(T item) { base.TryAdd(item); }  
    /// <summary>Gets an item from the pool.</summary>
    /// <returns>The removed or created item.</returns>
    /// <remarks>If the pool is empty, a new item will be created and returned.</remarks>
    public T GetObject()
    {
      T value;
      return base.TryTake(out value) ? value : _generator();
    }  
    /// <summary>Clears the object pool, returning all of the data that was in the pool.</summary>
    /// <returns>An array containing all of the elements in the pool.</returns>
    public T[] ToArrayAndClear()
    {
      var items = new List<T>();
      T value;
      while (base.TryTake(out value)) items.Add(value);
      return items.ToArray();
    }  
    protected override bool TryAdd(T item)
    {
      PutObject(item);
      return true;
    }  
    protected override bool TryTake(out T item)
    {
      item = GetObject();
      return true;
    }
  }
}
```

There's also a base class which looks like this:

```
/// <summary>
/// Provides a base implementation for producer-consumer collections that wrap other
/// producer-consumer collections.
/// </summary>
/// <typeparam name="T">Specifies the type of elements in the collection.</typeparam>
[Serializable]
public abstract class ProducerConsumerCollectionBase<T> : IProducerConsumerCollection<T>
{
private readonly IProducerConsumerCollection<T> _contained;  
/// <summary>Initializes the ProducerConsumerCollectionBase instance.</summary>
/// <param name="contained">The collection to be wrapped by this instance.</param>
protected ProducerConsumerCollectionBase(IProducerConsumerCollection<T> contained)
{
  if (contained == null) throw new ArgumentNullException("contained");
  _contained = contained;
}  
/// <summary>Gets the contained collection.</summary>
protected IProducerConsumerCollection<T> ContainedCollection { get { return _contained; } }  
/// <summary>Attempts to add the specified value to the end of the deque.</summary>
/// <param name="item">The item to add.</param>
/// <returns>true if the item could be added; otherwise, false.</returns>
protected virtual bool TryAdd(T item) { return _contained.TryAdd(item); }  
/// <summary>Attempts to remove and return an item from the collection.</summary>
/// <param name="item">
/// When this method returns, if the operation was successful, item contains the item removed. If
/// no item was available to be removed, the value is unspecified.
/// </param>
/// <returns>
/// true if an element was removed and returned from the collection; otherwise, false.
/// </returns>
protected virtual bool TryTake(out T item) { return _contained.TryTake(out item); }  
/// <summary>Attempts to add the specified value to the end of the deque.</summary>
/// <param name="item">The item to add.</param>
/// <returns>true if the item could be added; otherwise, false.</returns>
bool IProducerConsumerCollection<T>.TryAdd(T item) { return TryAdd(item); }  
/// <summary>Attempts to remove and return an item from the collection.</summary>
/// <param name="item">
/// When this method returns, if the operation was successful, item contains the item removed. If
/// no item was available to be removed, the value is unspecified.
/// </param>
/// <returns>
/// true if an element was removed and returned from the collection; otherwise, false.
/// </returns>
bool IProducerConsumerCollection<T>.TryTake(out T item) { return TryTake(out item); }  
/// <summary>Gets the number of elements contained in the collection.</summary>
public int Count { get { return _contained.Count; } }  
/// <summary>Creates an array containing the contents of the collection.</summary>
/// <returns>The array.</returns>
public T[] ToArray() { return _contained.ToArray(); }  
/// <summary>Copies the contents of the collection to an array.</summary>
/// <param name="array">The array to which the data should be copied.</param>
/// <param name="index">The starting index at which data should be copied.</param>
public void CopyTo(T[] array, int index) { _contained.CopyTo(array, index); }  
/// <summary>Copies the contents of the collection to an array.</summary>
/// <param name="array">The array to which the data should be copied.</param>
/// <param name="index">The starting index at which data should be copied.</param>
void ICollection.CopyTo(Array array, int index) { _contained.CopyTo(array, index); }  
/// <summary>Gets an enumerator for the collection.</summary>
/// <returns>An enumerator.</returns>
public IEnumerator<T> GetEnumerator() { return _contained.GetEnumerator(); }  
/// <summary>Gets an enumerator for the collection.</summary>
/// <returns>An enumerator.</returns>
IEnumerator IEnumerable.GetEnumerator() { return GetEnumerator(); }  
/// <summary>Gets whether the collection is synchronized.</summary>
bool ICollection.IsSynchronized { get { return _contained.IsSynchronized; } }  
/// <summary>Gets the synchronization root object for the collection.</summary>
object ICollection.SyncRoot { get { return _contained.SyncRoot; } }
```

**Wow!**  Thats a fair bit of code in C#, fair enough there is a lot of noise in the xml doc comments, but theres also a lot of boiler plate code in there too.

Ok now we have gotten that out of the way heres the good bit.  

Below is an agent based design which implements the same functionality but uses a lot less code.
    
```fsharp
module Poc

//Agent alias for MailboxProcessor
type Agent<'T> = MailboxProcessor<'T>
 
///One of three messages for our Object Pool agent
type PoolMessage<'a> =
  | Get of AsyncReplyChannel<'a>
  | Put of 'a * AsyncReplyChannel<unit>
  | Clear of AsyncReplyChannel<List<'a>>
 
/// Object pool representing a reusable pool of objects
type ObjectPool<'a>(generate: unit -> 'a, initialPoolCount) =
  let initial = List.init initialPoolCount (fun (x) -> generate())
  let agent = Agent.Start(fun inbox ->
    let rec loop(x) = async {
      let! msg = inbox.Receive()
      match msg with
      | Get(reply) ->
        let res = match x with
              | a :: b ->
                reply.Reply(a);b
              | [] as empty->
                reply.Reply(generate());empty
        return! loop(res)
      | Put(value, reply)->
        reply.Reply()
        return! loop(value :: x)
      | Clear(reply) ->
        reply.Reply(x)
        return! loop(List.empty<'a> )
    }
    loop(initial))  
  /// Clears the object pool, returning all of the data that was in the pool.
  member this.ToListAndClear() =
    agent.PostAndAsyncReply(Clear)
  /// Puts an item into the pool
  member this.Put(item) =
    agent.PostAndAsyncReply((fun ch -> Put(item, ch)))
  /// Gets an item from the pool or if there are none present use the generator
  member this.Get(item) =
    agent.PostAndAsyncReply(Get)
```

We have a discriminated union (PoolMessage) which describes the messages that we are going to use with this agent, they are pretty straight forward to follow.  

* **Get** simply returns either a stored item or generates a brand new one using the generator function which is passed into the ObjectPools constructor **(generate: unit -> 'a)**.  
* **Put** simply adds the item onto the internal list.  
* **Clear** simply returns the current pool and then clears it.

The core processing all happens in the **async{}** block, we simply wait for a message to arrive, then we pattern match on one of the messages either Get,Put, or Clear.

**Get** takes an item from the internal list if there are items present, otherwise it invokes the generator function and returns a newly generated object.  

For a **Put** operation we use the cons **(::)** operator to add the item onto the internal list via the recursive loop.

For the **Clear** operation we return the entire list then return an empty list to the recursive loop.

I think you will agree this is a nice succinct example of the flexibility and elegance of agents and yet another reason to use F# for more server side
activities.  It's not simply a language for the mathematical and finance orientated developers.

For anyone interested all of the code should be in my [GitHub repository ](http://bit.ly/mDQyfH )to download.

Thanks to [Tomas Petricek](http://tomasp.net/) for suggesting using the recursive loop to pass the list rather than using a ref cell and the (**:=**) operator.

Until next time...
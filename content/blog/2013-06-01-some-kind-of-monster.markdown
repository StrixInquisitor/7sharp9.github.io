---
author: 7sharp9
layout: post
title: "Some kind of monster"
date: 2013-06-01
comments: true
categories: [programming]
tags: [F#, Rx, Mono, C#]
description: ""
type: post
---
{{< figure class="img-left 4u" src="http://upload.wikimedia.org/wikipedia/en/2/29/Godzilla_%2754_design.jpg" >}}
What's 100 meters high and weighs in at around 60,000 tons?   No its not Godzilla, its Reactive extensions!

Lately on one of my projects I have been doing a lot of stream manipulation, and although I solved the problem quite easily using F# async workflows, there were other solutions available to help solve the problem.  I could of used things like [async await][2], [TPL Dataflow(TDF)][3], and [Reactive Extensions (Rx)][4].  This is going to be a short post on using Rx with F#.  <!-- more -->
### What is Rx?

Well for those of you that don't know anything about Rx I would suggest reading up a bit of the introduction material [here][4].  Here's a quick recap of what Rx is for everyone else:
> The Reactive Extensions (Rx) is a library for composing asynchronous and event-based programs using observable sequences and LINQ-style query operators. Using Rx, developers represent asynchronous data streams with Observables , query asynchronous data streams using LINQ operators , and parameterize the concurrency in the asynchronous data streams using Schedulers . Simply put, Rx = Observables + LINQ + Schedulers.

### Sample problem
Here's a sample problem:  I want to read from a file in buffered chunks and perform an action whenever a chunk is read.  As I said before there a many different ways to solve this problem but We'll use C# as a base to see how it could be done first in C#.  For this example We'll add an Extension extension method to the `Stream` class:

```csharp
public static class Extensions
{
   public static IObservable<byte[]> ToObservable (this Stream stream, int size)
   {
      return Observable.Create<byte[]> (observer =>
      {
         byte[] buffer = new byte[size];
         var deferedRead = Observable.Defer(() => stream.ReadAsync (buffer, 0, size).ToObservable());
         return Observable.Repeat(deferedRead)
                          .Select (i => buffer.Take(i).ToArray ())
                          .Subscribe (data => { if (data.Length > 0) observer.OnNext (data);
                                                else observer.OnCompleted ();
                                              }, observer.OnError, observer.OnCompleted);
      });
   }
}
```

Using this extension method we can then do something like this:

```csharp
var source = new FileStream (@"test.txt", FileMode.Open, FileAccess.Read);
source.ToObservable (16).Subscribe (_ => Console.WriteLine(_.Length));
```

This will print to the console the length of each chunk as it is read.

There are quite a few different Rx operators in this example, `Create`, `Defer`, `ToObservable`, `Repeat`, `Select`, and `Subscribe`.  Lets quickly go though the example and see what's going on.

First of all we create a custom observable sequence using `Observable.Create`.  This takes a lambda function with a single parameter `observer`, which is of type `IObserver<byte[]>`.  Using the `observer` we can produce elements in the sequence by using the methods `OnNext`, `OnError` and `OnCompleted`.

Next up we create a buffer to hold the data which will be read from the file in chunks.  This is just a simple array allocation `byte[] buffer = new byte[size];`

To allow us to consume the data from the file stream we can use the `ReadAsync` method which will return a `Task<byte[]>`.  There is an Rx extension method on `Task` called `ToObservable` so we use that too.  You will notice in the code that we are using `Observable.Defer`.  Why are we using that?  What would happen if we didn't?  Well, if we don't defer the `Task` for later execution and simply use to `Task.ToObservable()` we would be creating a new instance of the `Observable` sequence each time `ReadAsync` is called - This would mean we would have an infinite sequence comprised of the first chunk of the file, which isnt waht we want at all.  By using `Defer` we don't invoke the `Observable` `Task` until first subscription to the `Observable` sequence.

We use a fluent style to repeat the deferred Observable `deferedRead` using the `Repeat` method.

`Select` is now used to take the number of bytes from the `buffer`, we might have a stream which is not divisible by the buffer size which will mean that the last read will not be the size of the buffer.  In the lambda expression the `i` parameter is the number of the bytes returned from `ReadAsync`.

Finally we have `Subscribe`, this takes a lambda that is passed `data`, data being the current chunk or `byte[]`.  In the body of the lambda we check to see if we have received any bytes, if so then we call `observer.OnNext(data)` which creates the next element in the sequence.  If we didn't receive data then we call `observer.OnCompleted()`, which completes the sequence.  The last two parameters for `Subscribe` are the error and completed actions, we simply use the ones in the observer - `observer.OnError` and `observer.OnCompleted`.

If you read through the code again now it probably makes more sense the second time around, there's a lot of functionality squeezed into a small space but by using tried and tested components / functions in Rx you should have a better experience than rolling your own parts, of course you can do the same with [TDF][3] but I wont go into that here.

So what would all this look like in F#?

Well, if you try to do a direct port you start to hit a few issues due to the amount of overloads for some of the methods, `Zip` for example, has a staggering 19 overloads!!   This almost always means your working right at the edge of the ability of [type inferencing][6].  In order to determine what method overload you intended to use you have to add further type parameters, this can sometimes be a tricky business as F# lambda's are not always correctly typed back to `Action` and `Func`.

Lets see an example of that now:

```fsharp
module StreamExt =
   type Stream with
      member x.ToObservable(size) =
         Observable.Create(fun (observer: IObserver<_>) ->
            let buffer = Array.zeroCreate size
            let defered = Observable.Defer(fun () -> (x.ReadAsync (buffer, 0, size)).ToObservable())
            Observable.Repeat<int>(defered)
                      .Select(fun i -> buffer.Take(i).ToArray())
                      .Subscribe( (fun (data:byte[]) -> if data.Length > 0 then observer.OnNext(data)
                                                        else observer.OnCompleted()), observer.OnError, observer.OnCompleted ))
```

I had to add quite a few type annotations to get this working.  You can end up spending quite a while adding explicit types which isn't exactly an enjoyable or productive way of spending your time, sometimes you can hit a wall and have to annotate the function separately to see where the inference is failing.

To make things easier you can wrap the overloads with F# friendly versions.  In fact this has already been done in the [Fsharp.Reactive][5] repo on GitHub.  As I'm using Mono I had to do a quick compilation against the Reactive Extensions that come bundled with Mono 3.x rather than the nuget references.  I also added in a couple of function's that were missing from this version.   Here's the result usin `FSharp.Reactive`, I think you'll agree it looks a bit better and seems to flow quite nice with the pipeline operators in place.

```fsharp
module StreamExt =
   type Stream with
      member x.ToObservable(size) =
         Observable.Create (fun (observer: IObserver<_>) ->
            let buffer = Array.zeroCreate size
            Observable.Defer(fun () -> (x.ReadAsync (buffer, 0, size)).ToObservable())
            |> Observable.repeat
            |> Observable.map(fun i -> buffer |> Seq.take i |> Seq.toArray)
            |> Observable.subscribe(function
                                    | data when data.Length > 0 -> observer.OnNext(data)
                                    | _ -> observer.OnCompleted()) observer.OnError observer.OnCompleted)
```

Notable difference are the `Observable.Defer` is piped into `repeat`, `map` and `subscribe`.  Finally, the last piece that's different is the use of the Seq expression `(fun i -> buffer |> Seq.take i |> Seq.toArray)`  rather than the Linq `Take` function `.Select (i => buffer.Take(i).ToArray ())`.  To be honest there's not really much difference between the two, sequence expressions just seem more natural while using F#.  Lastly I switched from the if else expression to a pattern matching using the [function][7] keyword.  It's used in pattern matching when we want to match against only one parameter that's passed into the function.  This makes the subscribe function a little more compact.

Here is the details of the functions that were used in the above snippet so that you don't have to go looking in GitHub for details:
```fsharp
module Observable =
   ///Repeats the observable
   let repeat f = Observable.Repeat(source = f)

   /// maps the given observable with the given function
   let map f source = Observable.Select(source, Func<_,_>(f))

   /// Subscribes to the observable with all three callbacks
   let subscribe onNext onError onCompleted (observable: 'a IObservable) =
       observable.Subscribe(Observer.Create(Action<_> onNext, Action<_> onError, Action onCompleted))
```

I think that Rx is a very useful library but it's ironic that a functional programming oriented library is not easily usable from a functional language like F#.   There are over 400 Observable extension methods if you include all the overloads.  Its like ten thousand spoons when all you need is a knife! ...  Joking aside I wish the API designers had taken it easy when adding all the extension methods, when you are developing code the last thing you want to do is scroll up and down through method overloads trying to spot which exact overload you are looking for.

If you want some more samples you might want to take a look at [101 Rx Samples][0].  Also for reference when building something new, make sure you check out the [Reactive Extensions design Guide][1].  

Until next time!  

* * *
#### Essential listening:
*   Buckethead- The Elephant Man's Alarm Clock
*   Linkin Park - Hybrid Theory
*   Audioslave - Audioslave  
  
  {{< img-fit
    2u "http://upload.wikimedia.org/wikipedia/en/f/f5/Elephantalarmclock.jpg" "Buckethead- The Elephant Man's Alarm Clock"
    2u "http://upload.wikimedia.org/wikipedia/en/c/c9/Linkin_park_hybrid_theory.jpg"  "Linkin Park - Hybrid Theory"
    2u "http://upload.wikimedia.org/wikipedia/en/a/ac/Audioslave_-_Audioslave.jpg" "Audioslave - Audioslave" "" "" "" >}}

[0]: http://rxwiki.wikidot.com/101samples
[1]: http://go.microsoft.com/fwlink/?LinkID=205219
[2]: http://msdn.microsoft.com/en-us/library/vstudio/hh191443.aspx
[3]: http://msdn.microsoft.com/en-us/library/hh228603.aspx
[4]: http://rx.codeplex.com
[5]: https://github.com/fsharp/FSharp.Reactive
[6]: http://stackoverflow.com/a/501356/607275
[7]: http://msdn.microsoft.com/en-us/library/dd233242.aspx

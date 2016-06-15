---
author: 7sharp9
layout: post
title: "Monster Zero - Revisited"
date: 2013-06-05
comments: true
categories: [Programming]
tags: [F#, TDF, Mono, C#, Dataflow, TPL]
---
{% img left http://upload.wikimedia.org/wikipedia/en/c/ce/KingGhidorah.jpg 225 Monster Zero %}  
This creature is capable of tremendous destruction due to it's size, flight *(with the creature's wings also generating hurricane strength winds)* and possesses several breath weapons *(e.g., heat and energy)*.  

What am I talking about here?  Maybe it's [Monster Zero][0] or [King Ghidorah][0] as it's sometimes known.  No it's [TPL Dataflow][1]!  <!-- more -->

Yeah, yeah, I have a penchant for being over dramatic and writing quirky intros.  This post is about [TPL Dataflow][1] otherwise known as TDF.  I have blogged about this before in my [TDF agent series][2] but I thought it might be worth while returning to it while on the subject of monsters.  

The purpose of these posts is not to put you of using these libraries but to give you feel of how they might be used from an F# viewpoint.  Its not always easy to use these libraries even from C#, so jumping to another paradigm can sometimes leave you feeling bewildered, frustrated and lost.  

###What is TPL Dataflow

According to MSDN here's the description of TPL dataflow:  

>The Task Parallel Library (TPL) provides dataflow components to help increase the robustness of concurrency-enabled applications. These dataflow components are collectively referred to as the TPL Dataflow Library. This dataflow model promotes actor-based programming by providing in-process message passing for coarse-grained dataflow and pipelining tasks.  

Checkout the [MSDN][1] if you want to read a more in depth outline on TDF, or you could also refer to my [earlier posts][2] too.  

###Sample problem  

Here's a sample problem:  We have two documents that we want to use to collate a list of word occurrences, we then want to collect the results from both documents and print out the combined results.  

Although we're using TDF to solve this problem, we could of also used F# agents, Reactive Extensions, Linq, TPL, or a mix of all of those.  

We will stick to using the [F# REPL][10] for this post as it's fairly simple example and allows for a bit of interactivity.  Lets start by adding a reference, open up a few namespace's and read a couple of text files in.  In this instance we're going to use [Jane Eyre][3] by Charlotte Bronte, and [The Wendigo][8] by Algernon Blackwood, for no reason other than they were free to download and use as samples.  

{% codeblock lang:fsharp %}
#r "System.Threading.Tasks.Dataflow"
open System
open System.IO
open System.Threading.Tasks.Dataflow

let janeEyre = File.ReadAllText(@"Jane Eyre [Charlotte Bronte].txt")
let theWendigo = File.ReadAllText(@"The Wendigo [Algernon Blackwood].txt")
{% endcodeblock %}

The next thing we need to think about is how to count the words.  Let's create a recursive function that counts each occurrence of a passed in word against the full text.  The `wordCount` function recurses until `-1` is returned from the `IndexOf` function.  Before you argue about memory allocation, laziness etc, we're not interested in that at the moment, we just want to solve the problem at hand.  It would be fairly easy to split the input into a lazy sequence and iterate over it so we only keep a single line in memory.  

{% codeblock lang:fsharp %}
let wordCount (text: String) word =
   let rec loop position count =
      match text.IndexOf(word, position, StringComparison.InvariantCultureIgnoreCase) with
      | -1 -> count
      | i ->  loop (i + word.Length) (count + 1)
   loop 0 0
{% endcodeblock %}

Now we can start to create some TDF blocks, we shall create a [BroadcastBlock][4] first.  

>A BroadcastBlock provides a buffer for storing at most one element at time, overwriting each message with the next as it arrives.  

Messages are broadcast to all linked targets, all of which may consume a clone of the message.  

The lambda expression passed into the `BroadcastBlock` is it's clone function, in this case we will just use the string that is passed in: `fun s -> s`.  We will be using the `BroadcastBlock` to send the same message to multiple destinations later on.  

{% codeblock lang:fsharp %}
let broadcast = BroadcastBlock(fun s -> s)
{% endcodeblock %}

Now we will create a couple of [TransformBlocks][5].   

>A TransformBlock provides a dataflow block that invokes a provided Func(T, TResult) delegate for every data element received.  

The `TransformBlock` will accept a data element and transform it by invoking it's transform function.  Here we will [partially apply][11] the `wordCount` function by passing in the first parameter `text`.  This means that whenever the `TransformBlock` is passed data the `wordCount` function will already have the `text` parameter applied and will return the number of matches from the document.  The context of passed data here will be the word that we want to find.  We'll create one for Jane Eyre and one for The Wendigo:  

{% codeblock lang:fsharp %}
let transformJaneEyre = TransformBlock(wordCount janeEyre)
let transformTheWendigo  = TransformBlock(wordCount theWendigo)
{% endcodeblock %}  

Now that we have created three blocks we need to think about linking them together.  You can do this with the `LinkTo` method that every dataflow block has.  We could do that by calling the `LinkTo` methods as usual like this:  

{% codeblock lang:fsharp %}
broadcast.LinkTo(transformJaneEyre) |> ignore
broadcast.LinkTo(transformTheWendigo) |> ignore
{% endcodeblock %}  

That seems a little awkward, especially as we're not interested in the return parameter in this example, so instead we're going to create a little function to make this a little bit easier for ourselves:  

{% codeblock lang:fsharp %}
let (-->) source target = DataflowBlock.LinkTo(source, target) |> ignore
{% endcodeblock %}  

The `LinkTo` method returns an `IDisposable` that can be use to sever the link between the blocks that have just been joined.  There are also overloads of `LinkTo` that allow you to specify a predicate.  There is also an overload that takes a `DataflowLinkOptions` type which allows you to specify whether the new link is appended (`Append`), the maximum message that can be passed before the block is unlinked (`MaxMessages`), and finally and whether or not the completion of the former block is propagated to the latter (`PropagateCompletion`).  

We can now use the `-->` symbolic operator and use [infix notation][9] to link the blocks together.  Infix operators are expected to be placed between the two operands, which means we can define the links between the block like this:  

{% codeblock lang:fsharp %}
broadcast --> transformJaneEyre
broadcast --> transformTheWendigo
{% endcodeblock %}

Next we create a [JoinBlock][6].  

>A JoinBlock provides a dataflow block that joins across multiple dataflow sources, which are not necessarily of the same type, waiting for one item to arrive for each type before they’re all released together as a tuple that contains one item per type.

{% codeblock lang:fsharp %}
let join = JoinBlock<_,_,_>()

broadcast           --> join.Target1
transformJaneEyre   --> join.Target2
transformTheWendigo --> join.Target3
{% endcodeblock %}  

We create the `JoinBlock` then link the `broadcast`, `transformJaneEyre`, and `transformTheWendigo` to it.  This means that the `JoinBlock` will wait for data from **all** three blocks before sending the data on as a tuple of the three values.  

Finally we create the last block which is an [ActionBlock][7]

>An ActionBlock provides a dataflow block that invokes a provided Action(T) delegate for every data element received.

{% codeblock lang:fsharp %}
let writeOutput = 
    ActionBlock(fun(word:String, count1, count2) -> 
       Console.WriteLine("Word: {0}, Jane Eyre: {1}, The Wendigo: {2}", 
                         word.PadRight(10),
                         (string count1).PadLeft(3), 
                         (string count2).PadLeft(3) ) )
    
join --> writeOutput
{% endcodeblock %}

Right, that completes the hook up, now all that's left is to test it.  

{% codeblock lang:fsharp %}
let words = [|"cat";"cake";"anything";"laugh";"breeze";"hysterical";"ball";"them";"home";"bird"|]
for word in words do 
  broadcast.Post(word) |> ignore
{% endcodeblock %}

If we execute this then we we get the following output:

{% codeblock lang:bash %}
Word: cat       , Jane Eyre: 212, The Wendigo:  73
Word: cake      , Jane Eyre:  15, The Wendigo:   0
Word: anything  , Jane Eyre:  60, The Wendigo:  19
Word: laugh     , Jane Eyre:  68, The Wendigo:  17
Word: breeze    , Jane Eyre:  11, The Wendigo:   0
Word: hysterical, Jane Eyre:   1, The Wendigo:   1
Word: ball      , Jane Eyre:  11, The Wendigo:   1
Word: them      , Jane Eyre: 432, The Wendigo:  72
Word: home      , Jane Eyre:  90, The Wendigo:  11
Word: bird      , Jane Eyre:  35, The Wendigo:   0
{% endcodeblock %}

###Summary

From a conceptual viewpoint of view we're creating a BroadcastBlock which connects to two TransformBlocks.  The two TransformBlocks are then connected to the JoinBlock along with the BroadcastBlock.  Finally, the JoinBlock is connected to the ActionBlock.  

This creates a mini network where you can simply post a message to the input of the dataflow network and it will propagate through the network.  As with Reactive Extensions marble diagrams and pipeline diagrams are a great way to visualise the process flow.  

{% img https://lh5.googleusercontent.com/-r5TZAl6VERo/UbCwd4MXGTI/AAAAAAAABpg/MmQIb_0nwb8/w769-h218-no/Screen+Shot+2013-06-06+at+16.50.31.png 520 %}

I hope that sheds a little bit of light on how a dataflow network can be created with TDF.  Extremely complex behaviour's can be created by connecting up the simple dataflow building blocks, especially as the different block's can run with multiple degrees of parallelism an also run asynchronously using `Task<T>`.    

Until next time!  

* * *
####Essential listening:  
*   Alice In Chains - Unplugged  
*   Linkin Park - Meteora  
*   Soil - Scars  

{% img http://upload.wikimedia.org/wikipedia/en/4/43/AIC_Unplugged.jpg 125 Alice In Chains - Unplugged%}
{% img http://upload.wikimedia.org/wikipedia/en/6/64/MeteoraLP.jpg 125 Linkin Park - Meteora %}
{% img http://upload.wikimedia.org/wikipedia/en/4/44/Soilscars.jpg 125 Soil - Scars %}  

[0]: http://en.wikipedia.org/wiki/King_Ghidorah
[1]: http://msdn.microsoft.com/en-us/library/hh228603.aspx
[2]: http://moiraesoftware.com/blog/2012/01/22/FSharp-Dataflow-agents-I/
[3]: http://en.wikipedia.org/wiki/Jane_Eyre
[4]: http://msdn.microsoft.com/en-us/library/hh160447.aspx
[5]: http://msdn.microsoft.com/en-us/library/hh194782.aspx
[6]: http://msdn.microsoft.com/en-us/library/hh160286.aspx
[7]: http://msdn.microsoft.com/en-us/library/hh194684.aspx
[8]: http://en.wikipedia.org/wiki/Algernon_Blackwood#Novels
[9]: http://msdn.microsoft.com/en-us/library/dd233204.aspx
[10]: http://msdn.microsoft.com/en-us/library/dd233175.aspx
[11]: http://msdn.microsoft.com/en-us/library/dd233229.aspx

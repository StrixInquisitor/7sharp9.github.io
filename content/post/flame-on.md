+++
author = "7sharp9"
categories = ["programming"]
date = "2016-05-29T20:35:29+01:00"
description = "Producing FlameGraphs for F#"
linktitle = ""
title = "flame on"
type = "post"
tags = ["FSharp", "profiling", "FlameGraph"]

+++

A few weeks back I posted on Twitter that I was experimenting with flame graphs, In this post I will share how this was accomplished.  <!--more-->

{{< tweet 721829320646389762 >}}

### Requirements
First of all the requirements, I'm assuming your using a Mac just as I am.  If you are not then you might be able to use x-perf for windows check out [summarizing-xperf-cpu-usage-with-flame-graphs][3] for information in that area.  I don't really use Windows that often but if I do happen to try this out on Windows then I'll pop back here and update this post.

OK, so back to requirements, a Mac, Mono Installation, [Xcode][2] installed so that you can use [Instruments][1] to collect trace information, clone the FlameGraph repo itself:

```
git clone https://github.com/brendangregg/FlameGraph
```

The FlameGraph repo is a bunch of scripts to help process the trace data and produce an svg.  You can read more about FlameGraphs here: http://www.brendangregg.com/flamegraphs.html

### AOT the framework

Next step is to AOT compile all the mono runtime assemblies, you can do this by running the following commands from your mono installation.  For me this would be:
```
cd /Library/Frameworks/Mono.framework/Versions/4.4.0/lib/mono/
for i in `find gac -name '*dll'` */mscorlib.dll; do
   mono --aot $i
done
```

### AOT Your app
Now you need to AOT you applications files with the same command
```
mono --aot myApp.exe
```

### Attaching Instruments
Now you are ready to run your app and attach Instruments or have Instruments launch your app.  

For simplicity I opted to add this to the beginning of my app:
```fsharp
printfn "Press any key to start"
Console.ReadKey() |> ignore
```

That way I could just launch my app (noting the process id) and use the process browser within Instruments to attach.  
Now launch Instruments and select the Time Profiler template:

{{< figure src="/img/flame-on/timeprofiler-template.png" class="6u">}}

You can tweak the sampling interval with the settings on the right, in my example below I was using 40us because it was a really fast executing demo.  

Use the process browser in Instruments to choose the mono process running my app e.g. mono (16314).    

Now that instruments is attached hit the big red record button and hit any key on you app to start collecting data.  When you are finished just hit the top button in Instruments.  

You should end up with something similar to this, I used a few cycles of my 68000 emulator to get this data:

{{< figure src="/img/flame-on/trace.png" class="8u" >}}

### Exporting The Data 
Exporting the data is pretty easy, use expand all on a node in the collected data using `Cmd cLick`, you can also add filters to the data, I used `Atari` in the screen-shot above to constrain the output to nodes that contained `Atari`.  Now select export from the instrument menu:
{{< figure src="/img/flame-on/export.png" class="6u" >}}


### Producing the FlameGraph
Now you can open a move to the flamegraph repo that you cloned earlier and execute the following command replacing myoutput.csv|svg with your input/outputs.  

```
./stackcollapse-instruments.pl myoutput.csv | ./flamegraph.pl >myoutput.svg
```

You should now have a funky FlameGraph!

{{< figure src="/img/flame-on/68kcpusteps.svg" >}}

You could quite easily post process the csv output to clean up the mangled names that are a result of the AOT process.

Until next time ...

[1]:https://developer.apple.com/library/tvos/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/#//apple_ref/doc/uid/TP40004652-CH3-SW1
[2]:https://developer.apple.com/xcode/
[3]: https://randomascii.wordpress.com/2013/03/26/summarizing-xperf-cpu-usage-with-flame-graphs/


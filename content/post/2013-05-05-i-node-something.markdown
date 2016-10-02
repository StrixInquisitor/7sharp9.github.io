---
author: 7sharp9
layout: post
title: "I node something (Bout You)"
date: 2013-05-05
comments: true
categories:  [programming]
tags: [FSharp, node, async, lambda, CSharp, IronPython, edge, edge-fs, Azure]
description: ""
type: post
---
# What is Edge.js?
{{< figure class="img-left sixth" src="http://nodejs.org/images/logos/nodejs-dark.png" >}}  

Unless you *live in a hole* you have probably heard of [node.js][1] so I'll not bother to explain what it is or what it does.  An interesting project has come to light lately, namely [Edge.js][2].  The Edge.js project allows you to connect node.js with .Net.  

The creator of Edge.js Tomasz Janczuk sums this up nicely:  
  
  
>An edge connects two nodes  
This edge connects node.js with .NET

Currently Edge.js is only available on Windows but there is work underway to bring this to Mono, thus opening up the possibilities even further.  <!-- more -->

The coding model for Edge.js offers different integration options depending on the quantity of code you are writing, and whether you want to call a .Net dll directly.  

Here are a few examples:

## Single line lambda expressions:
```js
var edge = require('edge');
var hello = edge.func(
    'async (input) => { return ".NET welcomes " + input.ToString(); }'
);

hello('Node.js', function (error, result) {
    if (error) throw error;
    console.log(result);
});
```
## Multi line lambda expressions:
```js
var hello = require('edge').func(function () {/*
    async (input) => {
        return ".NET welcomes " + input.ToString(); 
    }
*/});

hello('Node.js', function (error, result) { ... });
```
## File based expresions:
```js
var hello = require('edge').func('hello.csx');

hello('Node.js', function (error, result) { ... });
```
## Invoking via a dll:
```js
var add7 = require('edge').func('My.Sample.dll');

add7(12, function (error, result) { ... }
```

The entry point into your .NET code is a delegate normalized to a `Func<object,Task<object>>`. This allows node.js code to call the .NET code asynchronously and avoid blocking the node.js event loop.  If you think about the possibilities of this for a moment, a lot of different options begin to open up with this framework.  I can foresee a lot of interesting things appearing in the future.  

There are currently two .Net compilers part of Edge.js.  A C# based compiler and an [IronPython][5] one.  You can probably guess what I'm going say next...   

# Introducing Edge-fs - An F# complier for edge.js

First let's look at the interop model for Edge.js:    
{{< figure class="6u" src="https://f.cloud.github.com/assets/822369/234085/b305625c-8768-11e2-8de0-e03ae98e7249.PNG" link="http://tjanczuk.github.io/edge/#/4" >}}

In summary, if we want to integrate with Edge.js then we must coerce whatever input that is passed to a single delegate function `Func<Object, Task<Object>>`

In terms of the C# Edge compiler a lambda expression is passed in the [async await][6] style:- 
```csharp
async (input) => { return ".NET welcomes " + input.ToString(); }
```

The Python Edge compiler is passed a lambda in it's native format too:

```
def hello(input):
        return "Python welcomes " + input

    lambda x: hello(x)
```

So where does that leave us with F# compiler support?  Well, I suppose the most intuitive support for F# would be to use F# async workflow support.  This would mean the that lambda expression would look like this:

```fsharp
fun input -> async{return ".NET welcomes " + input.ToString()}
```

You can see it's not that different from C#'s' async await style syntax, you can really see the F# async workflow heritage here.  
## Script Example  
Now lets look at how a script file or dll and have a look to see how this would fits:  

```fsharp
namespace global
type Startup() =
    let addSeven v =  v + 7
    member x.Invoke(input:obj) =
        let v = input :?> int
        async.Return (addSeven v :> obj) |> Async.StartAsTask
```

This is really easy too, the Async module has a StartAsTask function that perfectly fits here.  

By default Edge.js looks for a type in the global namespace called `Startup` with a public method called `Invoke`.  The invoke method takes a single parameter `input` which is of the type `Object`.  The return type of this method is as you might have guessed `Task<Object>`.  You can also add parameters to the node.js to indicate the location of the assembly, type and method name using the `assemblyName`, `typeName` and `methodName` parameters respectively.  

```js
var clrMethod = edge.func({
    assemblyFile: 'My.Edge.Samples.dll',
    typeName: 'Samples.FooBar.MyType',
    methodName: 'MyMethod'
});
```

## Further Documentation
Edge.js has some really good [documentation][4] so if your interested then you really should check it out.  I plan on supporting all of the calling conventions that the C# edge compiler has to offer.  At the moment only the in-line lambdas and the file based inputs have been tested, but I'm working on further examples, and fixes as needed.     

## Why do you need a Custom Compiler
As an aside, with dll based inputs any .Net language would work with Edge.js, you don't need a custom compiler.  The internals of Edge.js invoke your dll via reflection.  

```
Handle<v8::Value> ClrFunc::Initialize(const v8::Arguments& args)
{
    ...
    // reference .NET code through pre-compiled CLR assembly 
    String::Utf8Value assemblyFile(jsassemblyFile);
    String::Utf8Value nativeTypeName(options->Get(String::NewSymbol("typeName")));
    String::Utf8Value nativeMethodName(options->Get(String::NewSymbol("methodName")));  
    typeName = gcnew System::String(*nativeTypeName);
    methodName = gcnew System::String(*nativeMethodName);      
    assembly = Assembly::LoadFrom(gcnew System::String(*assemblyFile));
    ClrFuncReflectionWrap^ wrap = ClrFuncReflectionWrap::Create(assembly, typeName, methodName);
    result = ClrFunc::Initialize(
        gcnew System::Func<System::Object^,Task<System::Object^>^>(
            wrap, &ClrFuncReflectionWrap::Call));
    ...
```
A custom compiler is only required for compiling code in the form of scripts or lambda expressions.  It's expected that this will be a common use case so it's important to have a native F# compiler support.

So there we have it, a very quick whistle stop tour of **Edge-fs** the F# compiler for Edge.js.  I realise that this post only just skims the surface but I just wanted to get this out in the wild.  Ill be updating my [repo][7] over the next day or so, and a stable release will go out via the [npm package][8] as soon as things stabilise.   

Next time we're going to lift the lid on the F# Edge compiler and take a look at it's guts, we'll also go through some of the trials and tribulations I had along the way.  Ill also continue the series with some more documentation and samples too.  

Until next time!

{{< EssentialListening
  "|http://upload.wikimedia.org/wikipedia/en/4/43/Alice_In_Chains-Facelift.jpg|Alice In Chains - Facelift"
  "|http://upload.wikimedia.org/wikipedia/en/1/12/PanteraVulgarDisplayofPower.jpg|Pantera - Vulgar Display Of Power" >}}

[1]:http://nodejs.org
[2]:http://tjanczuk.github.io/edge/#/
[3]:http://tjanczuk.github.io/edge/#/4
[4]:https://github.com/tjanczuk/edge
[5]:http://ironpython.net
[6]:http://msdn.microsoft.com/en-us/library/vstudio/hh191443.aspx
[7]:https://github.com/7sharp9/edge-fs
[8]:https://npmjs.org

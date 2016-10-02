---
author: 7sharp9
title: "Meta-Matic"
layout: post
date: 2015-07-12
comments: true
categories: [programming]
tags: [FSharp, AST, Metaprogramming, Elixir]
slug: meta-matic
description: ""
type: post
---

Sit down, strap in, and prepare for take off, we're going Meta-Matic!  

We're going to be exploring some [metaprogramming][5] magic in F#.  Transforming 
its abstract syntax tree ([AST][1]) into another languages AST before executing 
it in another [virtual machine][2], Exciting!!<!--more-->

# Background
{{% blockquote footer="H.P Lovecraft" %}}
Nothing really known can continue to be acutely fascinating
{{% /blockquote %}}
First a little bit of background.  There are three main forms of metaprogramming 
in F#, although you could call [Type Providers][11] a form of metaprogramming albeit 
limited in scope.  I shall now briefly describe each below.  

## Quotations

According to MSDN:

{{% blockquote cite="https://msdn.microsoft.com/en-us/library/dd233212.aspx?f=255&MSPPError=-2147217396" %}}
Code quotations, a language feature that enables you to generate and work with 
F# code expressions programmatically. This feature lets you generate an abstract 
syntax tree that represent's F# code. The abstract syntax tree can then be traversed 
and processed according to the needs of your application
{{% /blockquote %}}

Quotations are fairly useful for translating some parts of the F# language to another 
language.  They can depend on [Reflection][6] and having compiled DLLs for any types 
referenced in the quoted expression.  They also have limitations like types are 
not supported, neither are variables that escape their quoted scope, and also 
generic definitions are not supported.  They cant really be used for staged 
metaprogramming where you may need to refer to the full language and its constructs.  

### Code Example
```fsharp
<@ [for i in 1 .. 10 do yield i * i] @>
```

A textual representation of what this looks like is as follows:
```
Quotations.Expr<int list> =
  Call (None, ToList,
    [Call (None, CreateSequence,
      [Call (None, Delay,
        [Lambda (unitVar,
          Call (None, Map,
            [Lambda (i, Call (None, op_Multiply, [i, i])),
              Call (None, op_Range,
                [Value (1), Value (10)])]))])])])
```
* * *
## Untyped AST
There's also the untyped [AST][1] that can be manipulated in some ways although 
it's not exactly pleasant to do.

### Code Example
```fsharp
[for i in 1 .. 10 do yield i * i]  
```

A textual representation of what this looks like is as follows:
```fsharp
[SynModuleOrNamespace
   ([Test
       {idRange = Test.fs (1,0--1,0);
        idText = "Test";}],true,
    [DoExpr
       (SequencePointAtBinding
          Test.fs (1,0--1,33)
            {...},
        ArrayOrListOfSeqExpr
          (false,
           CompExpr
             (true,{contents = true;},
              ForEach
                (SequencePointAtForLoop
                   Test.fs (1,1--1,20)
                     {...},
                 SeqExprOnly false,true,
                 Named
                   (Wild
                      Test.fs (1,5--1,6)
                        {...},
                    i {idRange = Test.fs (1,5--1,6);
                       idText = "i";},false,null,
                    Test.fs (1,5--1,6)
                      {...}),
                 App
                   (NonAtomic,false,
                    App
                      (NonAtomic,true,
                       Ident
                         op_Range
                           {idRange = Test.fs (1,12--1,14);
                            idText = "op_Range";},
                       Const
                         (Int32 1,
                          Test.fs (1,10--1,11)
                            {...}),
                       Test.fs (1,10--1,14)
                         {...}),
                    Const
                      (Int32 10,
                       Test.fs (1,15--1,17)
                         {...}),
                    Test.fs (1,10--1,17)
                      {...}),
                 YieldOrReturn
                   ((true, false),
                    App
                      (NonAtomic,false,
                       App
                         (NonAtomic,true,
                          Ident
                            op_Multiply
                              {idRange = Test.fs (1,29--1,30);
                               idText = "op_Multiply";},
                          Ident
                            i {idRange = Test.fs (1,27--1,28);
                               idText = "i";},
                          Test.fs (1,27--1,30)
                            {...}),
                       Ident
                         i {idRange = Test.fs (1,31--1,32);
                            idText = "i";},
  <snip>
```
As you can see it's a bit of a handful, I even omitted some of the range statements 
too.  It's possible to manipulate this but it's not exactly easy without a lot of 
helper functions and patience.  Maybe I can cover this in a future post as it has 
its own use case in terms of F# manipulation and recompilation, but that's a whole 
other story.  Outside of the compiler it's mainly used for code analysis type operations.  
* * *
## Typed AST / Typed Expressions
There's also the Typed AST (TAST) or Typed Expressions that fully represent's the 
F# language.  Like quotations, they give you a detailed view of checked/resolved/typed 
F# expressions.  They have no dependency on reflection and do not require any on 
disk assemblies.  They can be generated for solutions that that contain errors, 
and include all F# language constructs including those used in FSharp.Core.  

### Code Example
```fsharp
[for i in 1 .. 10 do yield i * i]  
```

A textual representation of what this looks like is as follows:
```fsharp
[Entity
  (Test,
    [InitAction
      Call (null,val toList,[],[int],
        [Call (null,val seq,[],[type int],
          [Call (null,val delay,[],[int],
            [Lambda (val unitVar,
              Call (null,val map,[],
                [type int; type int],
                  [Lambda (val i,
                    Call (null,val op_Multiply,[],
                      [type int; type int;
                        type int],[Value val i; Value val i]));
                          Call (null,val op_Range,[],[type int],
                            [Const (1,type int);
                              Const (10,type int)])]))])])])])] 
```

In this example and the preceding one you will notice that they both begin with 
either an `Entity` in this example, and `SynModuleOrNamespace` in the 
previous.  This is because dealing with the full F# metadata means we have access 
to the full range of F# syntactic constructs.  Entities also sit within another 
file based construct that I have omitted from these examples for the benefit of clarity.  
* * *
# Let's get to work 

For the purpose of this blog post we are going to use a basic F# module with a 
single function.  We will be performing all sorts of black magic so it's best to constrain 
this from the outset.  *(Note:  Im joking here)*

{{% blockquote footer="H.P. Lovecraft, letter to Frank Belknap Long" %}}
There is nothing more beautiful than an elegant mathematical proof. 
{{% /blockquote %}}
  
The module and function we will be using is very simple and looks like this:  

```fsharp
module Test
   let square x = x * x
```

## Retrieving the TAST  
First things first how do we get hold of the TAST?

We will need the `FSharp.Compiler.Service` package that can be installed from your 
IDE or favorite command line tool such as nuget:

```
nuget install FSharp.Compiler.Service
```

Now that it's installed we can start using it.  


```fsharp
#r "../packages/FSharp.Compiler.Service.0.0.89/lib/net45/FSharp.Compiler.Service.dll"
open System
open System.IO
open Microsoft.FSharp.Compiler.SourceCodeServices

let testModule = """module Test
   let square x = x * x
"""

let file = __SOURCE_DIRECTORY__ +  "/Test.fs"
File.WriteAllText(file, testModule)
let checker = FSharpChecker.Create(keepAssemblyContents=true)
let options =
  checker.GetProjectOptionsFromCommandLineArgs("Test", [|"-o:Test.dll";"-a";file|])
let checkProjectResults =
  checker.ParseAndCheckProject(options) |> Async.RunSynchronously
```
In order to examine the TAST we have to make sure that when creating the `FSharpChecker` 
we set the optional parameter `keepAssemblyContents` to true, this lets you examine 
the typed AST in full.  

## Examining the TAST
Examining `checkProjectResults` will yield several properties: `AssemblyContents`, 
`AssemblySignature`, `Errors`, `HasCriticalErrors`, and `ProjectContext`.  Most of 
those are pretty obvious, if we examine `Errors` and `HasCriticalErrors` we could 
decide to abandon further processing but for the purposes of this example we are 
going to assume that there are no errors.  

The property that we are interested in is `AssemblyContents` this essentially 
contains a list of `ImplementationFiles`, and an `ImplementationFile` has a list of 
`FSharpImplementationFileDeclaration`.  

Let's go ahead and examine what we currently have: 
```
checkProjectResults.AssemblyContents.ImplementationFiles.Head.Declarations
```
This will yield a similar structure to what you saw above in the list comprehension.   
I have shortened the the `Microsoft.FSharp.Core.*` prefixes to save space. 
```fsharp
[Entity
 (Test,
  [MemberOrFunctionOrValue
     (val square,[[val x]],
      Call
        (null,val op_Multiply,[],
         [type int; type int;
          type int],[Value val x; Value val x]))])]
```

The `FSharpImplementationFileDeclaration` is a [discriminated union][7] type that is defined as follows:

```fsharp
type FSharpImplementationFileDeclaration = 
  | Entity of FSharpEntity * FSharpImplementationFileDeclaration list
  | MemberOrFunctionOrValue of FSharpMemberOrFunctionOrValue * FSharpMemberOrFunctionOrValue list list * FSharpExpr
  | InitAction of FSharpExpr
```

With this in mind we could now traverse the tree with something like this:

```fsharp
let rec processDecl decl =
  match decl with
  | Entity(ent, declList) ->
      printfn "Entity"
      declList |> List.iter processDecl
  | InitAction(expr) -> printfn "Init Action"
  | MemberOrFunctionOrValue(memb, curriedParameterGroups, expr)->
      printfn "Member"
      printfn "Expressions: %A" expr

for implFile in checkProjectResults.AssemblyContents.ImplementationFiles do
  for decl in implFile.Declarations do
    processDecl decl
```
   
This would yield similar output to what we saw before although we are now getting a
feel for the structure of the tree.
```
Entity:Test
Member:square
Expressions: Call
  (null,val op_Multiply,[],
   [type int; type int;
    type int],[Value val x; Value val x])
```
* * *
# Preparing To transform

Ok, so we know a little about the TAST structure and how to navigate it, now we 
can look to transform it.  So what can we transform the F# TAST to?  

Absolutely anything we want, but for the purposes of this article let's choose 
something interesting ... like Elixir.  

## Elixir language
So what is Elixir?  I'll quote the description from the [Elixir website][8] as this 
post is not about describing the Elixir language.  
{{% blockquote %}}
Elixir is a dynamic, functional language designed for building scalable and 
maintainable applications.
{{% /blockquote %}}

Elixir leverages the Erlang VM, known for running low-latency, distributed and 
fault-tolerant systems, while also being successfully used in web development and 
the embedded software domain.
{% endblockquote %}

## Elixir AST structure
The Elixir AST structure is described like this:
```elixir
{atom | tuple, list, list | atom}
```
*  The first element is an atom or another tuple in the same representation;
*  The second element is a keyword list containing metadata, like numbers and contexts
*  The third element is either a list of arguments for the function call or an 
atom. When this element is an atom, it means the tuple represent's a variable.

Elixir is a quite capable language and I will be exploring it further in future 
posts.  For now we'll simply transform the equivalent function in Elixir to 
its native AST to see what it looks like.  This is really quite easy in Elixir 
as it has a far more natural metaprogramming experience than F#.  We just use 
`quote(opts, block)`, where `block` is the expression we want to get the AST 
representation for.  As an example let's `quote` the Elixir equivalent of the F# 
`Test` module with the `square` function:  

```elixir
quote do
  defmodule Test do
    def square(x) do
      x * x
    end
  end
end
```
This results in an Elixir AST:
```elixir
{:defmodule, [context: Elixir, import: Kernel],
 [{:__aliases__, [alias: false], [:Test]},
  [do: {:def, [context: Elixir, import: Kernel],
    [{:square, [context: Elixir], [{:x, [], Elixir}]},
     [do: {:*, [context: Elixir, import: Kernel],
       [{:x, [], Elixir}, {:x, [], Elixir}]}]]}]]}
```
# A Tale Of Two Trees
Let's start defining a tree structure for our transformation, we are going to be 
transforming from an F# TAST straight to an Elixir AST as Elixir has capabilities 
to use the AST fairly easily, we don't have to specifically work with the AST 
to get it back to code.  

```fsharp
type ElixirAst =
  | Fragment of atom : string * metadata : (string * string ) list * args : Arguments
  | Nested of (string * Arguments) list
  
and Arguments =
  | Binding of string 
  | Bindings of list<string>
  | Arguments of list<ElixirAst>
```

We use a recursive [Discriminated Union][9] that describes the 
Elixir AST.  You should be able to see it's made up of either a `Fragment` or a 
`Nested` both of those can contain `Arguments` that can in turn contain `Binding`, 
`Bindings`, or a list of `Arguments` that can also be `Fragment` or `Nested` ...  

It's quite a brain twister at first glance but it should hopefully make sense.  

## Transforming The Trees

We are now going to write a pair of functions that will iterate over the F# TAST 
and produce an Elixir AST, its more or less a case of transforming the F# constructs to the Elixir equivalents.

```fsharp
let rec traverseExpr expr =
  match expr with
  | BasicPatterns.Call(_expr,functionMemberOrVal,_,_typeSig, expressions) ->
      let argumentFragments = expressions |> List.map traverseExpr
      let functionName = functionMap functionMemberOrVal.LogicalName
      Fragment(functionName, ["context:", "Elixir";"import:", "Kernel"], Arguments argumentFragments)
  | BasicPatterns.Value(v) -> Fragment(":" + v.DisplayName, [], Binding "Elixir")
  | other -> failwithf "Not implmented: %A" other

let rec traverse decl = 
  match decl with
  | FSharpImplementationFileDeclaration.Entity(ent, declList) ->
      let arguments =
        [yield Fragment(":__aliases__", ["alias:", "false"], Bindings[":" + ent.DisplayName])
         let declList = declList |> List.map traverse
         yield Nested[("do:", Arguments declList)] ]
      let moduledef =
        Fragment(":defmodule", ["context:", "Elixir"; "import:", "Kernel"], Arguments arguments )
      moduledef
      
  | FSharpImplementationFileDeclaration -> failwith "Not implemented"
  
  | FSharpImplementationFileDeclaration.MemberOrFunctionOrValue(memb, curriedParameterGroups, expr)->
      let args = 
        [ let parameters =
            [for group in curriedParameterGroups do
               for param in group do
                 yield ":" + param.DisplayName] |> List.map (fun name -> Fragment(name, [], Binding "Elixir"))
        
          yield Fragment(":" + memb.DisplayName, ["context:", "Elixir"], Arguments parameters)
          yield Nested[("do:", Arguments [traverseExpr expr] ) ] ]
      Fragment(":def", ["context:", "Elixir"; "import:", "Kernel"], Arguments args)
```
Notice the `FSharpImplementationFileDeclaration` fails with not implemented as 
it's not needed in this example so we will omit it here.

One thing I haven't shown in detail in the F# TAST section was the expressions, 
that's because there are 43 different elements that make up F#'s' full typed 
expressions.  In our example we only need to use `Call` and `Value` so the 
other expression types fail with not implemented too.

Let's try this out by pumping the F# TAST through our new function and see what we get:

```fsharp
let astElements =
  [for implFile in checkProjectResults.AssemblyContents.ImplementationFiles do
    for decl in implFile.Declarations do
      yield traverse decl]
```

```
[Fragment
 (":defmodule",[("context:", "Elixir"); ("import:", "Kernel")],
  Arguments
    [Fragment (":__aliases__",[("alias:", "false")],Bindings [":Test"]);
     Nested
       [("do:",
         Arguments
           [Fragment
              (":def",[("context:", "Elixir"); ("import:", "Kernel")],
               Arguments
                 [Fragment
                    (":square",[("context:", "Elixir")],
                     Arguments [Fragment (":x",[],Binding "Elixir")]);
                      Nested
                        [("do:",
                          Arguments
                            [Fragment
                               (":*",
                                [("context:", "Elixir"); ("import:", "Kernel")],
                                Arguments
                                  [Fragment (":x",[],Binding "Elixir");
                                   Fragment (":x",[],Binding "Elixir")])])]])])]])]

```
{{% blockquote footer="H.P. Lovecraft, From Beyond" %}}
"I have harnessed the shadows that stride from world to world to sow death and madness."
{{% /blockquote %}}

## Stringify the AST

Ok, so now that we have an F# based tree structure that describes the Elixir AST we 
now need it back in stringly form to do anything with it.  We could of just traversed 
the tree writing the AST elements but that wouldn't of given us the flexibility to 
traverse the tree in the future.  

As we will be working with string's and `StringBuilder`s let's define a few helper functions:

```fsharp
let (++) (ctx:StringBuilder) (str:String) = ctx.Append(str)
let (+>) (ctx:StringBuilder) (f:StringBuilder -> StringBuilder) = f ctx
```

These will let us pipe two `StringBuilder` append operations together and also allow 
us to pipe into other functions that take a `StringBuilder` as an argument.  

First we'll define something simple that will just render the metadata:
```fsharp
let printMeta (meta: (string * string) list) (sb:StringBuilder) =
  meta
  |> Seq.map (fun (key,value) -> sprintf "%s %s" key value)
  |> String.concat ", "
  |> Printf.bprintf sb "[%s]"
  sb
```

Now a duo of functions that will traverse the `ElixirAst`, processing the 
`Arguments`, `Fragment`, and `Nested` elements:

```fsharp
let rec printArgs args (sb:StringBuilder) =
  match args with
  | Binding s -> sb ++ s
  | Bindings xs -> sb ++ sprintf "[%s]" (String.concat ", " xs)
  | Arguments args ->
    sb ++ "[" |> ignore
    args |> List.iteri (fun i item -> printAst item sb |> ignore
                                      if i < args.Length-1 then sb ++ ", " |> ignore)
    sb ++ "]"

and printAst ast (sb:StringBuilder) =
  match ast with
  | Fragment(atom, metadata, args) ->
    sb ++ sprintf "{%s, " atom +> printMeta metadata ++ ", " +> printArgs args ++ "}"
  | Nested(items) ->
    sb ++ "[" |> ignore

    items
    |> List.iteri (fun i (name, args) ->
                     sb ++ sprintf "%s " name +> printArgs args |> ignore
                     if i < items.Length-1 then sb ++ ", " |> ignore)

    sb ++ "]"
```

Finally capture the results by iterating over the AST and collecting the 
stringified results:
```fsharp
let elixirAstItems = 
  let sb = StringBuilder()
  [for element in astElements do
     printAst element sb |> ignore
     yield sb.ToString()]
```

# VM execution

{{% blockquote footer="H.P. Lovecraft, The Case of Charles Dexter Ward" %}}
"I have brought to light a monstrous abnormality, but I did it for the sake of 
knowledge. Now for the sake of all life and Nature you must help me thrust it 
back into the dark again."
{{% /blockquote %}}

## Helper Process
We'll go for the simplest option here: we will create a new process, start up an 
interactive Elixir session, and send across a few commands including the AST that 
we just created.  

Now we'll define a type to help us with this:

```fsharp
type public ProcessWrapper(file: string, env:Collections.Generic.IDictionary<_,_>) =
  let info = ProcessStartInfo(RedirectStandardInput = true,
                              RedirectStandardOutput = true,
                              UseShellExecute = false,
                              CreateNoWindow = true,
                              FileName = file)

  do 
    for item in env do
      info.EnvironmentVariables.[item.Key] <- item.Value
  let proc = new Process(StartInfo = info)
  let outputBuffer = StringBuilder()
  let dataReceived = proc.OutputDataReceived.Subscribe(fun data -> outputBuffer.AppendLine(data.Data) |> ignore)

  [<CLIEvent>]
  member this.OutputReceived = proc.OutputDataReceived

  [<CLIEvent>]
  member this.ErrorReceived = proc.ErrorDataReceived

  member this.Start() =
    proc.Start() |> ignore
    proc.BeginOutputReadLine()

  member this.Send(line: string) =
    proc.StandardInput.WriteLine(line)

  member this.Flush() =
    proc.StandardInput.Flush()

  member x.GetOutputBuffer(?clear) =
    let data = outputBuffer.ToString()
    if clear.IsSome then outputBuffer.Clear() |> ignore
    data

  interface IDisposable with
    member x.Dispose() =
      dataReceived.Dispose()
      proc.Dispose()
```
## Code Execution
Now that we have all of the pieces in place let's start the process and add some 
path variables so that iex can find the things that are normally in your path.  

```fsharp
let iex =
  new ProcessWrapper("/usr/local/Cellar/elixir/1.0.4/bin/iex",
                     dict["PATH", "/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin"])
iex.Start()
```

Now we can send across out AST definition and assign it to `mycode`

```fsharp
iex.Send("mycode =" + List.head elixirAstItems)
```

If we examined the output of iex now we can see that this has been assigned:

```fsharp
iex.GetOutputBuffer()
```
```elixir
{:defmodule, [context: Elixir, import: Kernel],
 [{:__aliases__, [alias: false], [:Test]},
  [do: [{:def, [context: Elixir, import: Kernel],
     [{:square, [context: Elixir], [{:x, [], Elixir}]},
      [do: [{:*, [context: Elixir, import: Kernel],
         [{:x, [], Elixir}, {:x, [], Elixir}]}]]]}]]]}
```

Now we can use Elixir's metaprogramming capabilities to compile this AST so we can use it:

```fsharp
iex.Send("mycode |> Code.compile_quoted()")
```
```elixir
[{Test, <<70, 79, 82, 49, 0, 0, 4, 104, 66, 69, 65, 77, 69, 120, 68, 99, 0 ...>>}]
```
Yes!!  It looks like our AST has now been compiled, we have a module named `Test` 
with binary data assigned.  Now let's see if we can execute our `square` function:

```fsharp
iex.Send("Test.square(16)")
```
```elixir
[256]
```

Woo-hoo job done!  I hope you enjoyed this whistle stop tour through the 
[Mountains Of Madness][10].  

I don't know about you, but I need a cup of tea!

Until next time ...
  
[1]:https://en.wikipedia.org/wiki/Abstract_syntax_tree
[2]:https://en.wikipedia.org/wiki/Virtual_machine
[4]:https://en.wikipedia.org/wiki/H._P._Lovecraft
[5]:https://en.wikipedia.org/wiki/Metaprogramming
[6]:https://msdn.microsoft.com/en-us/library/f7ykdhsy(v=vs.110).aspx
[7]:https://msdn.microsoft.com/en-us/library/dd233226.aspx
[8]:http://elixir-lang.org
[9]:https://msdn.microsoft.com/en-us/library/dd233226.aspx
[10]:https://en.wikipedia.org/wiki/At_the_Mountains_of_Madness
[11]:https://msdn.microsoft.com/en-us/library/hh156509.aspx

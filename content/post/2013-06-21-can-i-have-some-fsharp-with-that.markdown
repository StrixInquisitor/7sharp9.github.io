---
author: 7sharp9
layout: post
title: "Can I have some F# with that?"
date: 2013-06-21
comments: true
categories: [programming]
tags: [FSharp, CSharp, ScriptCs, Roslyn, scripting]
description: ""
type: post
---
{{< figure class="img-left sixth" src="http://scriptcs.net/images/logo@2x.png" >}} There's been a fair bit of activity lately from a project called [ScriptCS][1], it allows you to put together a project using C# as a lightweight scripting language, forgoing the use of Visual Studio which can sometimes be too bloated and bulky.  

It also allows you to use C# in a Read Evaluate Print Loop - [REPL][2].  This is nothing new to F# and indeed lots of other languages have REPL's too.  One of the other benefits of ScriptCs is that it also integrates nicely with Nuget allowing you to use your favourite libraries quite easily.  Finally there are Script Sacks which can be used to further reduce the amount of code you need to write when working with common frameworks.  

It would be nice to leverage some of this new functionality from F#, and I don't like to see F# left out, especially when F# already has a REPL environment and is a really great language for scripting.  
<!-- more -->
The F# compiler is also open source so we can utilize the code to add various tooling and features like refactoring, formatting, and analysis.  See [Fantomas][3], [FSharp-Refactor][4], and the [FSharpBinding][5] for more details.  
- - -
### Lets get to work

The interface for adding a new script engines looks like this:

```
public interface IScriptEngine
{
    string BaseDirectory { get; set; }
    ScriptResult Execute(string code, 
                         string[] scriptArgs, 
                         IEnumerable<string> references, 
                         IEnumerable<string> namespaces, 
                         ScriptPackSession scriptPackSession);
}
```

First lets create a new namespace and open up all the namespaces we need:  

```fsharp
namespace ScriptCs.Engine.FSharp
open ScriptCs
open Common.Logging
open System
open System.IO
open System.Collections.Generic
open Microsoft.FSharp.Compiler.Interactive.Shell
open ExtCore
open System.Linq
```

Next we need to store the result of an attempted expression evaluation.  When we send code to an interactive session we might be in one of three different states:

   * Incomplete - We have entereed a line but the expression is not complete, in F# we use `;;` to indicate the end of an expression.  
   * Error - The entered expression resulted in an error.
   * Success - The expression that was entered was evaluated successfully.  

We can model this with a [discriminated union][6] like this:  

```fsharp
type Result = 
| Success of String 
| Error of string 
| Incomplete
```

Next we will augment the existing [FsiEvaluationSession][7] type to allow it to be encapsulated and used by our interface.  

```fsharp
type FSharpEngine(host:ScriptHost) = 
   let stdin        = new StreamReader(System.IO.Stream.Null)
   let stdoutStream = new CompilerOutputStream()
   let stdout       = StreamWriter.Synchronized(new StreamWriter(stdoutStream, AutoFlush=true))
   let stderrStream = new CompilerOutputStream()
   let stderr       = StreamWriter.Synchronized(new StreamWriter(stderrStream, AutoFlush=true))
    
   let getOutput (session: FsiEvaluationSession) code = 

      let tryget() = 
         let error = stderrStream.Read()
         if error.Length > 0 then Error(error) else
         Success(stdoutStream.Read())

      try session.EvalInteraction(code)
          if code.EndsWith ";;" then tryget() 
          else Incomplete
      with ex -> Error ex.Message
        
   let commonOptions = [| "fsi.exe"; "--nologo"; "--readline-"|]
   let session = FsiEvaluationSession(commonOptions, stdin, stdout, stderr)

   let (>>=) (d1:#IDisposable) (d2:#IDisposable) = 
      {new IDisposable with member x.Dispose() = d1.Dispose(); d2.Dispose()}

   member x.Execute(code) = 
      getOutput session code

   member x.AddReference(ref) =
      session.EvalInteraction(sprintf "#r @\"%s\"" ref)

   member x.SilentAddReference(ref) = 
      x.AddReference(ref)
      stdoutStream.Read() |> ignore

   member x.ImportNamespace(namespace') =
      session.EvalInteraction(sprintf "open %s" namespace')

   member x.SilentImportNamespace(namespace') =
      x.ImportNamespace(namespace')
      stdoutStream.Read() |> ignore

   interface IDisposable with
      member x.Dispose() =
         (stdin >>= stdoutStream >>= stdout >>= stderrStream >>= stderr).Dispose()
```

We are mainly just wrapping the `FsiEvaluationSession` here, adding convenience methods to evaluate code and gather the output from the compiler streams.  The current open source implementation of `FsiEvaluationSession` uses streams to add input and receive the output and errors.  Stream processing makes sense when you are just dealing with a Console with in, out, and error streams, but it gets decidedly more complex if you want deterministic evaluation.  Stream observation, polling, and looking for termination characters is fairly awkward to get right.  

After a few conversations with [Don Syme][15] he kindly assisted with an experimental version of `FsiEvaluationSession` that allowed expressions to be evaluated using the `EvalInteraction` and `EvalExpression` functions rather than writing directly to the input stream.  I'm very grateful for the work Don has done so far to help me with this.  I think more work on hosted compilation will result in a lot of very useful tools and techniques.  

You can also see in this section that I was also playing around with the symbolic operator `>>=` to compose together all the disposable streams at once.  I suppose the inspiration for this (If you can call it that) is [Reactive Extensions][16] which has a [CompositeDisposable][8], and also the disposable computation builder that [Tomas Petricek][11] made available as [fssnip][10].  The [object expression][12] that I used here seemed like a sensible option, and also shows the usefulness of object expressions.   

Finally we implement the interface using the `FSharpEngine` as the type to be stored in the scripting session:  

```fsharp
type  FSharpScriptEngine( scriptHostFactory:IScriptHostFactory, logger: ILog) =
   let mutable baseDir = String.empty
   let [<Literal>]sessionKey = "F# Session"
   
   interface IScriptEngine with
      member x.BaseDirectory with get() = baseDir and set value = baseDir <- value
      member x.Execute(code, args, references, namespaces, scriptPackSession) =
         let distinctReferences = references.Union(scriptPackSession.References).Distinct()
         let sessionState = 
            match scriptPackSession.State.TryGetValue sessionKey with
            | false, _ -> let host = scriptHostFactory.CreateScriptHost(ScriptPackManager(scriptPackSession.Contexts), args)
                          logger.Debug("Creating session")
                          let session = new FSharpEngine(host)
                 
                          distinctReferences |> Seq.iter (fun ref -> logger.DebugFormat("Adding reference to {0}", ref)
                                                                     session.SilentAddReference ref )
                 
                          namespaces.Union(scriptPackSession.Namespaces).Distinct() 
                          |> Seq.iter (fun ns -> logger.DebugFormat("Importing namespace {0}", ns)
                                                 session.SilentImportNamespace ns)
                 
                          let sessionState = SessionState<_>(References = distinctReferences, Session = session)
                          scriptPackSession.State.Add(sessionKey, sessionState)
                          sessionState 
            | true, res -> logger.Debug("Reusing existing session") 
                           let sessionState = res :?> SessionState<FSharpEngine>
                           
                           let newReferences = match sessionState.References with
                                               | null -> distinctReferences
                                               | refs when Seq.isEmpty refs -> distinctReferences
                                               | refs ->  distinctReferences.Except refs
                           newReferences |> Seq.iter (fun ref -> logger.DebugFormat("Adding reference to {0}", ref)
                                                                 sessionState.Session.AddReference ref ) 
                           sessionState      

         match sessionState.Session.Execute(code) with
         | Success result -> let cleaned = 
                                result.Split([|"\r"; "\n"|], StringSplitOptions.RemoveEmptyEntries)
                                |> Array.filter (fun str -> not(str = "> "))
                                |> String.concat "\r\n"
                             ScriptResult(ReturnValue = cleaned)
         | Error e -> ScriptResult(CompileException = exn e )
         | Incomplete -> ScriptResult()
```

For the most part `Execute` is a simple port of the Roslyn implementation, mainly due to the way ScriptCs is currently implemented.  There is a preprocessor that amongst other things parses reference additions `(#r)`, passing them down to the `Execute` method.  I think eventually a registrable command plugin for ScriptCs will appear that will make custom REPL commands easy to add and configure.  

Any new references are added in this snippet, where we leverage pattern matching.  

```fsharp
let newReferences = match sessionState.References with
                    | null -> distinctReferences
                    | refs when Seq.isEmpty refs -> distinctReferences
                    | refs ->  distinctReferences.Except refs
newReferences |> Seq.iter (fun ref -> logger.DebugFormat("Adding reference to {0}", ref)
                                      sessionState.Session.AddReference ref )
```

Also of note is the final matching block `match sessionState.Session.Execute(code) with`  we use pattern matching against the discriminated union that is returned by `Session.Execute(code)`.  If `Execute` returns a `Success` we do a bit of a clean up on the result.  We split the string based on carriage returns and newlines, filter out any prompts `> `, then reassemble the sting using `String.Concat`.  We put this into the `Result` property of a `ScriptResult`.  I do actually have a version of `FsiEvaluationSession` that suppresses prompts but I've not merged that in yet.  An `Error` results in the `CompileException` property being used on the `ScriptResult`.  Finally if the expression is `Incomplete` we don't output a result or display an error as we are waiting for more input, we simply return an empty `ScriptResult`.  

```fsharp
match sessionState.Session.Execute(code) with
| Success result -> let cleaned = 
                       result.Split([|"\r"; "\n"|], StringSplitOptions.RemoveEmptyEntries)
                       |> Array.filter (fun str -> not(str = "> "))
                       |> String.concat "\r\n"
                    ScriptResult(ReturnValue = cleaned)
| Error e -> ScriptResult(CompileException = exn e )
| Incomplete -> ScriptResult()
```

The final part is to plug this into ScriptSC, we do this by changing the `Initialize` method of `CompositionRoot`, all we need to do is Register our engine rather than the `RoslynScriptEngine` one:  

```
builder.RegisterType<ScriptExecutor>().As<IScriptExecutor>();
builder.RegisterType<RoslynScriptEngine>().As<IScriptEngine>();
```

To this:
```
builder.RegisterType<ScriptExecutor>().As<IScriptExecutor>();
builder.RegisterType<FSharpScriptEngine>().As<IScriptEngine>();
```

And there we have it, hack session complete!  

Several thing are missing from my implementation, namely debug support and script pack support.  The guys over at ScriptCs are continuing to evolve the API to allow plugins like this to work properly.  Multi-line support should be coming soon, if you run a REPL session using F# then a prompt is added when you hit return.  There is also a GitHub issue raised to [add runtime packs for plugging in different runtimes/languages][13], this will pave the way for ScriptCs to be available on Mono too *(If you disable the Roslyn based project and only use the F# Engine then it does actually work on Mono now.)*.   

Once these issues are resolved in ScriptCs then hopefully F# will become a simple language plugin or even be merged into ScriptCs itself.  

You can find my GitHub repository [here][14], feel free to hack away, add issues, pull requests are welcome too!

Until next time!  

{{< EssentialListening
    "|http://upload.wikimedia.org/wikipedia/en/8/81/The_Smashing_Pumpkins_-_Oceania_cover.jpg|Smashing Pumpkins - Oceania"
    "|http://upload.wikimedia.org/wikipedia/en/d/dc/Megadeth-RustInPeace.jpg|Megadeth - Rust In Peace" >}}

[1]: http://scriptcs.net
[2]: http://en.wikipedia.org/wiki/Read–eval–print_loop
[3]: https://github.com/dungpa/fantomas
[4]: https://github.com/Lewix/fsharp-refactor
[5]: https://github.com/fsharp/fsharpbinding
[6]: http://msdn.microsoft.com/en-us/library/dd233226.aspx
[7]: https://github.com/fsharp/fsharp/blob/master/src/fsharp/fsi/fsi.fs#l2219
[8]: http://msdn.microsoft.com/en-us/library/system.reactive.disposables.compositedisposable(v=vs.103).aspx
[9]: http://www.fssnip.net/2s
[10]: http://www.fssnip.net
[11]: http://tomasp.net/blog/
[12]: http://msdn.microsoft.com/en-us/library/dd233237.aspx
[13]: https://github.com/scriptcs/scriptcs/issues/346
[14]: https://github.com/7sharp9/scriptcs
[15]: http://blogs.msdn.com/b/dsyme/
[16]: http://msdn.microsoft.com/en-us/data/gg577609.aspx

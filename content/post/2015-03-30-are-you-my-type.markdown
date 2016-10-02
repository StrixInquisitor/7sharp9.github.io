---
author: 7sharp9
layout: post
title: "Are you my type?"
date: 2015-03-30
comments: true
categories: [programming]
tags: [FSharp, Types]
description: ""
type: post
---
Did you know there was more to the type matching operator than just pattern matching and exception handling?
<!--more-->

The type matching operator is defined thus: `:?`  It can be used by [pattern matching][1] to match on a specific type.  For example, you might want to test that an object is a certain type or deal with an object being one of several different types.  Pattern matching on types is your friend here:

```fsharp
match symbolUse.Symbol with
| :? FSharpMemberOrFunctionOrValue
| :? FSharpUnionCase
| :? FSharpEntity
| :? FSharpField
| :? FSharpGenericParameter
| :? FSharpActivePatternCase
| :? FSharpParameter
| :? FSharpStaticParameter ->
    match getSymbolDeclarationLocation symbolUse currentFile solution with
    | SymbolDeclarationLocation.External -> false
    | SymbolDeclarationLocation.Unknown -> false
    | _ -> true
| _ -> false
```

During pattern matching you can also use the `as` assignment operator to assign a named binding to the match so you can use it directly.  This is somewhat akin to using `is` and `as` in C#, or using an `as` and then a `null` check.  Yuck!  None of that kind of thing in F#:

```fsharp
let isPrivateToFile = 
    match symbolUse.Symbol with
    | :? FSharpMemberOrFunctionOrValue as m -> not m.IsModuleValueOrMember
    | :? FSharpEntity as m -> m.Accessibility.IsPrivate
    | :? FSharpGenericParameter -> true
    | :? FSharpUnionCase as m -> m.Accessibility.IsPrivate
    | :? FSharpField as m -> m.Accessibility.IsPrivate
    | _ -> false
```

It can also be used in [exception handing][2] to match a specific type of exception, as in this example where `TimeoutExceptions` are caught:

```fsharp
member x.GetDeclarationSymbols(line, col, lineStr) = 
    match infoOpt with 
    | None -> None
    | Some (checkResults, parseResults) -> 
        let longName,residue = Parsing.findLongIdentsAndResidue(col, lineStr)
        // Get items & generate output
        try
            let results = 
                Async.RunSynchronously (checkResults.GetDeclarationListSymbols(Some parseResults, line, col, lineStr, longName, residue, fun _ -> false), timeout = ServiceSettings.blockingTimeout )
            Some (results, residue)
        with :? TimeoutException -> None
```

A final use for `:?` that people either don't tend to use or know about, is during a normal expression assignment.  In this example `item :? DotNetProject` would evaluate to true when `item` is a `DotNetProject`.  

```fsharp
override x.SupportsItem(item:IBuildTarget) =
    item :? DotNetProject
```

Although not used that often I find the `:?` operator to be really useful.  

As usual F# helps to keep things short, succinct, and sweet! 

Until next time!

[1]:https://msdn.microsoft.com/en-gb/library/dd547125.aspx
[2]:https://msdn.microsoft.com/en-us/library/dd233194.aspx

{{< EssentialListening
     "|http://upload.wikimedia.org/wikipedia/en/5/51/Artofrebellioncover.JPG|Suicidal Tendencies - The Art Of Rebellion"
     "|http://upload.wikimedia.org/wikipedia/en/7/78/Riseagainsttheblackmarket.jpg|Rise Against - The Black Market" >}}
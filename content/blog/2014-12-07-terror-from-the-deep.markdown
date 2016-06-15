---
author: 7sharp9
layout: post
title: "Terror From The Deep"
date: 2014-12-10
comments: true
categories: [programming]
tags: [F#, MonoDevelop,Xamarin Studio, FSharp.Compiler.Services]
description: ""
type: post
---

{{< figure class="img-left" src="https://dl.dropboxusercontent.com/s/29mripr5z1ik5tr/tftd.png?dl=0"  >}}

So this is my Christmas special.  I've been asked on numerous times to write about the F# addin for Xamarin studio which is in the [fsharpbinding repo][1], this repo is shared with the emacs support and also the Sublime Text support.  So in this edition we will be taking a deep dive into the terrifying deep depths of the F# compiler and F# addin development...
<!--more-->
Actually I'm only joking, adding features to the F# compiler and F# addin is fairly easy depending on what you want to do.  You can run into issues along the way which means you might need to delve into more of the F# compilers functionality, essentially to derive and adapt new functions that you might need.  

What I'm going to show is how to add a new autocompletion list where instead of a standard completion list, its categorised by the type that the methods are derived from.  As an example you would be able to see the `ToString` methods etc on the `Obj` type andy other methods defined on their particular derived type.  

## One door leads to the source
Lets have a look at the current completion list function in the F# compiler:

```fsharp
member GetDeclarationsAlternate : 
	ParsedFileResultsOpt:ParseFileResults option * 
	line: int * 
	colAtEndOfPartialName: int * 
	lineText:string * qualifyingNames: 
	string list * 
	partialName: string * 
	?hasTextChangedSinceLastTypecheck: (obj * range -> bool) 
	-> Async<DeclarationSet>
```

Essentially this function takes a lot of parameters, I don't want to go into the details too much as the [FCS sample pages][2] does a good job of that.  So what is a `DeclarationSet`?  Well as you expect its a collection of `Declarations`.  A Declaration has a `Glyph`, `Name`, and `DescriptionText`.  The `DescriptionText` is a `ToolTipText` which is a text based rendering of the declaration in question.  

### ToolTipElement
```fsharp
/// Describe a comment as either a block of text or a file+signature reference into an intellidoc file.
type XmlComment =
| XmlCommentNone
| XmlCommentText of string
| XmlCommentSignature of (*File and Signature*) string * string

/// A single data tip display element
type ToolTipElement = 
| ToolTipElementNone
/// A single type, method, etc with comment.
| ToolTipElement of (* text *) string * XmlComment
// /// A parameter of a method.
// | ToolTipElementParameter of string * XmlComment * string
/// For example, a method overload group.
| ToolTipElementGroup of ((* text *) string * XmlComment) list
/// An error occurred formatting this element
| ToolTipElementCompositionError of string
 ```

At first glance this information is quite interesting but in use the limitation of text based rendering becomes apparent.  How can you break down the information into easily renderable parts or know the underlying types that make up the declaration.  Text based manipulation means a lot of work, and also lots of potential bugs, as you would expect with text based or weakly typed system.

Lets have a look at the `GetDeclarationsAlternate` function and see if we have access to any detailed information:

### GetDeclarationsAlternate
```fsharp
    member info.GetDeclarationsAlternate(parseResultsOpt, line, colAtEndOfNamesAndResidue, lineStr, qualifyingNames, partialName, ?hasTextChangedSinceLastTypecheck) = 
        let hasTextChangedSinceLastTypecheck = defaultArg hasTextChangedSinceLastTypecheck (fun _ -> false)
        reactorOp DeclarationSet.Empty (fun scope -> scope.GetDeclarations(parseResultsOpt, line, lineStr, colAtEndOfNamesAndResidue, qualifyingNames, partialName, hasTextChangedSinceLastTypecheck))
 ```

 OK, so that just calls `GetDeclarations` after doing a check for changes since the last type check, lets go deeper...

### GetDeclarations
```fsharp
    member x.GetDeclarations (parseResultsOpt:ParseFileResults option, line, lineStr, colAtEndOfNamesAndResidue, qualifyingNames, partialName, hasTextChangedSinceLastTypecheck) : DeclarationSet =
        let isInterfaceFile = SourceFileImpl.IsInterfaceFile mainInputFileName
        ErrorScope.Protect 
            Range.range0 
            (fun () -> 
                match GetDeclItemsForNamesAtPosition(parseResultsOpt, Some qualifyingNames, Some partialName, line, lineStr, colAtEndOfNamesAndResidue, ResolveTypeNamesToCtors, ResolveOverloads.Yes, hasTextChangedSinceLastTypecheck) with
                | None -> DeclarationSet.Empty  
                | Some(items,denv,m) -> 
                    let items = items |> filterIntellisenseCompletionsBasedOnParseContext (parseResultsOpt |> Option.bind (fun x -> x.ParseTree)) (mkPos line colAtEndOfNamesAndResidue)
                    let items = if isInterfaceFile then items |> List.filter IsValidSignatureFileItem else items
                    DeclarationSet.Create(infoReader,m,denv,items,reactorOps,checkAlive))
            (fun msg -> DeclarationSet.Error msg)
 ```

 Right, this is more interesting, if we look at the pattern match `match filterIntellisenseCompletionsBasedOnParseContext` you can see we have there are `items`, `denv`, and `m`.  Now what exactly is an `Item`?  

 Lets go deeper still and take a look....

### Item
```fsharp
/// Represents an item that results from name resolution
type Item = 
    /// Represents the resolution of a name to an F# value or function.
    | Value of  ValRef
    /// Represents the resolution of a name to an F# union case.
    | UnionCase of UnionCaseInfo
    /// Represents the resolution of a name to an F# active pattern result.
    | ActivePatternResult of ActivePatternInfo * TType * int  * range
    /// Represents the resolution of a name to an F# active pattern case within the body of an active pattern.
    | ActivePatternCase of ActivePatternElemRef 
    /// Represents the resolution of a name to an F# exception definition.
    | ExnCase of TyconRef 
    /// Represents the resolution of a name to an F# record field.
    | RecdField of RecdFieldInfo

    // The following are never in the items table but are valid results of binding an identitifer in different circumstances. 

    /// Represents the resolution of a name at the point of its own definition.
    | NewDef of Ident
    /// Represents the resolution of a name to a .NET field 
    | ILField of ILFieldInfo
    /// Represents the resolution of a name to an event
    | Event of EventInfo
    /// Represents the resolution of a name to a property
    | Property of string * PropInfo list
    /// Represents the resolution of a name to a group of methods
    | MethodGroup of string * MethInfo list
    /// Represents the resolution of a name to a constructor
    | CtorGroup of string * MethInfo list
    /// Represents the resolution of a name to the fake constructor simulated for an interface type.
    | FakeInterfaceCtor of TType
    /// Represents the resolution of a name to a delegate
    | DelegateCtor of TType
    /// Represents the resolution of a name to a group of types
    | Types of string * TType list
    /// CustomOperation(nm, helpText, methInfo)
    /// Used to indicate the availability or resolution of a custom query operation such as 'sortBy' or 'where' in computation expression syntax
    | CustomOperation of string * (unit -> string option) * MethInfo option
    /// Represents the resolution of a name to a custom builder in the F# computation expression syntax
    | CustomBuilder of string * ValRef
    /// Represents the resolution of a name to a type variable
    | TypeVar of string * Typar
    /// Represents the resolution of a name to a module or namespace
    | ModuleOrNamespaces of Tast.ModuleOrNamespaceRef list
    /// Represents the resolution of a name to an operator
    | ImplicitOp of Ident * TraitConstraintSln option ref
    /// Represents the resolution of a name to a named argument
    | ArgName of Ident * TType * ArgumentContainer option
    /// Represents the resolution of a name to a named property setter
    | SetterArg of Ident * Item 
    /// Represents the potential resolution of an unqualified name to a type.
    | UnqualifiedType of TyconRef list
```

Finally lets look at the `Create` function of `DeclarationSet` to see what's involved:

### DeclarationSet - Create
```fsharp
// Make a 'Declarations' object for a set of selected items
static member Create(infoReader:InfoReader, m, denv, items, reactor, checkAlive) = 
    let g = infoReader.g
     
    let items = items |> RemoveExplicitlySuppressed g
    
    // Sort by name. For things with the same name, 
    //     - show types with fewer generic parameters first
    //     - show types before over other related items - they usually have very useful XmlDocs 
    let items = 
        items |> List.sortBy (fun d -> 
            let n = 
                match d with  
                | Item.Types (_,(TType_app(tcref,_) :: _)) -> 1 + tcref.TyparsNoRange.Length
                // Put delegate ctors after types, sorted by #typars. RemoveDuplicateItems will remove FakeInterfaceCtor and DelegateCtor if an earlier type is also reported with this name
                | Item.FakeInterfaceCtor (TType_app(tcref,_)) 
                | Item.DelegateCtor (TType_app(tcref,_)) -> 1000 + tcref.TyparsNoRange.Length
                // Put type ctors after types, sorted by #typars. RemoveDuplicateItems will remove DefaultStructCtors if a type is also reported with this name
                | Item.CtorGroup (_, (cinfo :: _)) -> 1000 + 10 * (tcrefOfAppTy g cinfo.EnclosingType).TyparsNoRange.Length 
                | _ -> 0
            (d.DisplayName,n))

    // Remove all duplicates. We've put the types first, so this removes the DelegateCtor and DefaultStructCtor's.
    let items = items |> RemoveDuplicateItems g

    if verbose then dprintf "service.ml: mkDecls: %d found groups after filtering\n" (List.length items); 

    // Group by display name
    let items = items |> List.groupBy (fun d -> d.DisplayName) 

    // Filter out operators (and list)
    let items = 
        // Check whether this item looks like an operator.
        let isOpItem(nm,item) = 
            match item with 
            | [Item.Value _]
            | [Item.MethodGroup(_,[_])] -> 
                (IsOpName nm) && nm.[0]='(' && nm.[nm.Length-1]=')'
            | [Item.UnionCase _] -> IsOpName nm
            | _ -> false              
        let isFSharpList nm = (nm = "[]") // list shows up as a Type and a UnionCase, only such entity with a symbolic name, but want to filter out of intellisense
        items |> List.filter (fun (nm,items) -> not (isOpItem(nm,items)) && not(isFSharpList nm)) 


    let decls = 
        // Filter out duplicate names
        items |> List.map (fun (nm,itemsWithSameName) -> 
            match itemsWithSameName with
            | [] -> failwith "Unexpected empty bag"
            | items -> 
                new Declaration(nm, GlyphOfItem(denv,items.Head), Choice1Of2 (items, infoReader, m, denv, reactor, checkAlive)))

    new DeclarationSet(Array.ofList decls)
```

This looks very promising, this information could be just what we need.  If we do a quick search and see what else uses `Items` so we can get a better idea of how its used.  Lets just see if there are any pattern matches for Item.Value to get a quick idea:

*   nameres.fs
*   tc.fs
*   fsi.fs
*   service.fs
*   ServiceDeclarations.fs
*   Symbols.fs

## You're a symbol for your kind
The matches in `Symbol.fs` look interesting, you can see it's relatively easy to construct a `Symbol` if you have access to the relevant parts.  Having a list of symbols available rather than a `DeclarationSet` of `ToolTipElement` could be just what we need.

Lets look at constructing a symbol rather than the declaration set:

```fsharp
member x.GetDeclarationListSymbols (parseResultsOpt:FSharpParseFileResults option, line, lineStr, colAtEndOfNamesAndResidue, qualifyingNames, partialName, hasTextChangedSinceLastTypecheck) =
    let isInterfaceFile = SourceFileImpl.IsInterfaceFile mainInputFileName
    ErrorScope.Protect 
        Range.range0 
        (fun () -> 
            match GetDeclItemsForNamesAtPosition(parseResultsOpt, Some qualifyingNames, Some partialName, line, lineStr, colAtEndOfNamesAndResidue, ResolveTypeNamesToCtors, ResolveOverloads.Yes, hasTextChangedSinceLastTypecheck) with
            | None -> List.Empty  
            | Some(items,_denv,_m) -> 
                let items = items |> filterIntellisenseCompletionsBasedOnParseContext (parseResultsOpt |> Option.bind (fun x -> x.ParseTree)) (mkPos line colAtEndOfNamesAndResidue)
                let items = if isInterfaceFile then items |> List.filter IsValidSignatureFileItem else items

                //do filtering like Declarationset
                let items = items |> RemoveExplicitlySuppressed g
                
                // Sort by name. For things with the same name, 
                //     - show types with fewer generic parameters first
                //     - show types before over other related items - they usually have very useful XmlDocs 
                let items = 
                    items |> List.sortBy (fun d -> 
                        let n = 
                            match d with  
                            | Item.Types (_,(TType_app(tcref,_) :: _)) -> 1 + tcref.TyparsNoRange.Length
                            // Put delegate ctors after types, sorted by #typars. RemoveDuplicateItems will remove FakeInterfaceCtor and DelegateCtor if an earlier type is also reported with this name
                            | Item.FakeInterfaceCtor (TType_app(tcref,_)) 
                            | Item.DelegateCtor (TType_app(tcref,_)) -> 1000 + tcref.TyparsNoRange.Length
                            // Put type ctors after types, sorted by #typars. RemoveDuplicateItems will remove DefaultStructCtors if a type is also reported with this name
                            | Item.CtorGroup (_, (cinfo :: _)) -> 1000 + 10 * (tcrefOfAppTy g cinfo.EnclosingType).TyparsNoRange.Length 
                            | _ -> 0
                        (d.DisplayName,n))

                // Remove all duplicates. We've put the types first, so this removes the DelegateCtor and DefaultStructCtor's.
                let items = items |> RemoveDuplicateItems g

                if verbose then dprintf "service.ml: mkDecls: %d found groups after filtering\n" (List.length items); 

                // Group by display name
                let items = items |> List.groupBy (fun d -> d.DisplayName) 

                // Filter out operators (and list)
                let items = 
                    // Check whether this item looks like an operator.
                    let isOpItem(nm,item) = 
                        match item with 
                        | [Item.Value _]
                        | [Item.MethodGroup(_,[_])] -> 
                            (IsOpName nm) && nm.[0]='(' && nm.[nm.Length-1]=')'
                        | [Item.UnionCase _] -> IsOpName nm
                        | _ -> false              
                    let isFSharpList nm = (nm = "[]") // list shows up as a Type and a UnionCase, only such entity with a symbolic name, but want to filter out of intellisense
                    items |> List.filter (fun (nm,items) -> not (isOpItem(nm,items)) && not(isFSharpList nm)) 

                let items = 
                    // Filter out duplicate names
                    items |> List.map (fun (_nm,itemsWithSameName) -> 
                        match itemsWithSameName with
                        | [] -> failwith "Unexpected empty bag"
                        | items ->
                            items 
                            |> List.map (fun item -> let symbol = FSharpSymbol.Create(g, thisCcu, tcImports, item)
                                                     FSharpSymbolUse(g, _denv, symbol, ItemOccurence.Use, _m)))

                //end filtering
                items)
        (fun _msg -> [])
```

Looking at the code you can see its various pieces *cobbled* together to construct an `FSharpSymbolUSe` rather than a `DeclarationSet`.  This should allow us to create a more elaborate autocompletion which displays members by base type rather than a flat list.

## There Are No Flowers in the Real World…

So that's the easy bit done, now over to MonoDevelop.  We need to rip out the old completions and splice in the new one, currently it's defined in `FSharpTextEditorCompletion` and `FSharpMemberCompletionData`.  

Lets have a look at `CompletionData` which we will need to recreate for our purposes:

### CompletionData

```fsharp
type CompletionData
	abstract member Icon : IconId with get, set
	abstract member DisplayText : string with get, set
	abstract member Description : string with get, set
	abstract member CompletionText : string with get, set
	abstract member GetDisplayDescription : bool -> string
	abstract member GetRightSideDescription : bool -> string
	abstract member CompletionCategory : CompletionCategory with get, set
	abstract member DisplayFlags : DisplayFlags with get, set
	abstract member CreateTooltipInformation : bool -> TooltipInformation
	abstract member HasOverloads : () -> bool
	abstract member OverloadedData : () -> IEnumerable<ICompletionData>
	abstract member AddOverload : ICompletionData -> () 
	abstract member InsertCompletionText : CompletionListWindow * ref KeyActions * Gdk.Key * char * Gdk.ModifierType -> ()
	abstract member CompareTo : obj -> int
```

All of these are virtual in the `CompletionData` type, what we will need to do is add overrides for the `HasOverloads`, `OverloadedData`, `AddOverload`, and `CreateTooltipInformation` to give us the functionality we require.  It's going to be vety similar to the old code except we will be using symbols rather than `ToolTipElement` data to create the completion data.  

Lets create a new `FSharpMemberCompletionData`:  

```fsharp
type internal FSharpMemberCompletionDataSorted(name, icon, symbol:FSharpSymbol, overloads:FSharpSymbol seq) =
    inherit CompletionData(CompletionText = Lexhelp.Keywords.QuoteIdentifierIfNeeded name, 
                           DisplayText = name, 
                           DisplayFlags = DisplayFlags.DescriptionHasMarkup,
                           Icon = icon)

    /// Check if the datatip has multiple overloads
    override x.HasOverloads = not (Seq.isEmpty overloads)

    /// Split apart the elements into separate overloads
    override x.OverloadedData =
        overloads
        |> Seq.map (fun symbol -> FSharpMemberCompletionDataSorted(symbol.DisplayName, icon, symbol, Seq.empty) :> _ )

    override x.AddOverload (data: ICompletionData) = ()

    override x.CreateTooltipInformation (smartWrap: bool) = 
      let tip = SymbolTooltips.getTooltipFromSymbol symbol FSharpDisplayContext.Empty None
      match tip  with
      | ToolTips.ToolTip (signature, xmldoc) ->
            let toolTipInfo = new TooltipInformation(SignatureMarkup = signature)
            match xmldoc with
            | Full(summary) -> toolTipInfo.SummaryMarkup <- summary
                               toolTipInfo
            | Lookup(key, potentialFilename) ->
                let summary = 
                    maybe {let! filename = potentialFilename
                           let! markup = TipFormatter.findDocForEntity(filename, key)
                           let summary = Tooltips.getTooltip Styles.simpleMarkup markup
                           return summary }
                summary |> Option.iter (fun summary -> toolTipInfo.SummaryMarkup <- summary)
                toolTipInfo
            | EmptyDoc -> toolTipInfo
      | _ -> TooltipInformation()
```  

In this section you can see the use of the `maybe` computation expression *(You wont find the 'M' word mentioned here thank you very much!)* to simplify the creation of the `Lookup` tooltip's.  `Lookup` means pulling the information from [monodoc][3] which loads the xmldoc files, and `Full` means there is xmldoc's present in the compiler.  `Full` will occur in your own files and `Lookup` will occur in referenced assemblies.  

We also need a define little type to hold the category as the `CompletionCategory` type is abstract:

```fsharp
type Category(category) =
    inherit CompletionCategory(category, null)
    override x.CompareTo other = compare x.DisplayText other.DisplayText
```

Next we will add a function called `getCompletionData` to the existing `FSharpTextEditorCompletion`.  

```fsharp
let getCompletionData (symbols:FSharpSymbol list list) =

    let categories = Dictionary<string, Category>()

    let getOrAddCategory id =
        let found, item = categories.TryGetValue id
        if found then item
        else let cat = Category id 
             categories.Add (id,cat)
             cat

    let (|Function|Val|Unknown|) (symbol:FSharpSymbol) =
      match symbol with
      | MemberOrFunctionOrValue symbol
          when not (isConstructor symbol) ->
              if symbol.FullType.IsFunctionType && not symbol.IsPropertyGetterMethod && not symbol.IsPropertySetterMethod 
              then Function symbol                         
              else Val symbol
      | _ -> Unknown symbol

    let symbolToIcon (s:FSharpSymbol) = 
        match s with
        | ActivePatternCase _ -> Stock.Enum
        | Field _ -> Stock.Field
        | UnionCase _ -> Stock.Enum
        | Class -> Stock.Class
        | Delegate -> Stock.Delegate
        | Event -> Stock.Event
        | Property -> Stock.Property
        | Function _ -> MStock.Method
        | Val _ -> Stock.Field
        | Enum -> Stock.Enum
        | Interface -> Stock.Interface
        | Module -> Stock.Class
        | Namespace -> Stock.NameSpace
        | Record -> Stock.Class
        | Union -> Stock.Enum
        | ValueType -> Stock.Struct
        | _ -> Stock.Struct

    let symbolToCompletionData (symbol:FSharpSymbol) =
       let cd = FSharpMemberCompletionDataSorted(symbol.Head.DisplayName, symbolToIcon symbol.Head, symbol.Head, symbol.Tail)
       match symbol.Head with
       | :? FSharpMemberOrFunctionOrValue as func ->
           d.CompletionCategory <- getOrAddCategory func.EnclosingEntity.FullName
       | other ->
       cd.CompletionCategory <- getOrAddCategory (other.FullName.Substring (0, other.FullName.LastIndexOf '.'))

    symbols
    |> List.map symbolToCompletionData
    :> ICompletionData)
```

We have a few helper function's here, `getOrAddCategory` to get or add categories.  An active pattern `(|Function|Val|Unknown|)` to help to split `MemberOrFunctionOrValue` into `Function`, `Val` or `Unknown` sub types.  `symbolToIcon` to get a stock icon to represent the different types of item that will appear in the completion list.  And finally we have a map function, `symbolToCompletionData` which uses all of the other helper functions to project each symbol into a new `FSharpMemberCompletionDataSorted`.  This is done by using either `func.EnclosingEntity.FullName` if the type match is `FSharpMemberOrFunctionOrValue` or `other.FullName.Substring (0, other.FullName.LastIndexOf '.')` if the type match is anything else.  

You can see that the symbols are mapped using `List.map` and `symbolToCompletionData` at the end of the function.  The resulting `FSharpMemberCompletionDataSorted` is finally coerced into an `ICompletionData` with the `:>` operator.  

Finally all that's left is to change `x.CodeCompletionCommandImpl` in `FSharpTextEditorCompletion`, all we need to do is change the match statement to use the functions we defined above:  

```fsharp
match tyRes.GetDeclarations(line, col, lineStr) with
| Some(decls, residue) when decls.Items.Any() ->
      let items = decls.Items
                  |> Array.map (fun mi -> FSharpMemberCompletionData(mi) :> ICompletionData)
      result.AddRange(items)
| _ -> ()
```

To use the new `GetDeclarationSymbols` function:  

```fsharp
match tyRes.GetDeclarationSymbols(line, col, lineStr) with
| Some (symbols, residue) -> result.AddRange (getCompletionData symbols)
| None -> ()
```

Phew! I think we are done.  Spinning up Xamarin Studio with the new addin shows the new completion list:  
{{< figure src="https://dl.dropboxusercontent.com/s/hkojvf87qyuxvla/completion.png?dl=0" >}}  

We now have completion list sorted by the inheritor, which is especially nice for displaying members on hierarchical API's.  As a little bonus pressing `Shift Up/Down` will also move between the categories.

See, that wasn't so scary was it?

Until next time!

* * *  
#### Essential Christmas listening:  
*   Megadeth - Peace Sells... But Who's Buying?  
*   Megadeth - So far, so good... so what!  
*   Megadeth - Rust in Peace  
*   Exodus - Fabulous Disaster  

{{< img-fit
2u "http://upload.wikimedia.org/wikipedia/en/4/40/Megadeth_-_Peace_Sells..._But_Who%27s_Buying-.jpg" "Megadeth - Peace Sells... But Who's Buying"
2u "http://upload.wikimedia.org/wikipedia/en/7/7f/Megadeth-SoFar.jpg" "Megadeth - So far, so good, so what"
2u "http://upload.wikimedia.org/wikipedia/en/d/dc/Megadeth-RustInPeace.jpg" "Megadeth - Rust in Peace"
2u "http://upload.wikimedia.org/wikipedia/en/7/72/Exodus_-_Fabulous_Disaster.jpg" "Exodus - Fabulous Disaster" "" "" >}}


[1]: https://github.com/fsharp/fsharpbinding
[2]: http://fsharp.github.io/FSharp.Compiler.Service/editor.html#Getting-auto-complete-lists
[3]: http://www.mono-project.com/docs/tools+libraries/tools/monodoc/

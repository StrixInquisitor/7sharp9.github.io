---
author: 7sharp9
layout: post
title: "Flux Compression (redux)"
date: 2014-11-05
comments: true
categories: [programming]
tags: [TypeProviders,FSharp]
description: ""
type: post
---

First of all the title, redux because I'm revising post I started on earlier in the year, compression because this has to do with compression, and Flux, which is also part of the redux, one of the first things I remember writing on the net was an article about Flux Compression Generators on [H2G2][1], its still there too!
<!-- more --> 
This was a post I started writing back in January that I never got round to finishing.  

Once upon a time I had a need to quickly browse a zip file and it's Crc, so I quickly put together a [Type Provider][2] as a way to help in this en-devour.  I'm going to split the code into a few section and run a commentary over each block so you can see what I did and why.

# Zip Provider
I'm going to use [SharpCompress][3] as the basis for peering into zip files, you could also choose any other zip API.  Essentially to open and peruse a zip the API consists of the following:  

```fsharp
let zipfile = SharpCompress.Archive.ArchiveFactory.Open(fileName)

for entry in zipFile.Entries do
	...
```

This gives us the ability to open a zip file and to iterate over its contents via a sequence of `IArchiveEntry`

## Creating the Type Provider

To create a Type Provider we need to create a type which looks like this, also notice the `TypeProviderAssembly` attribute:  

```fsharp
[<TypeProvider>]
type public ZipProvider(cfg : TypeProviderConfig) as this =
    inherit TypeProviderForNamespaces()

    let asm = Assembly.GetExecutingAssembly()
    let ns = "Xebec"
    let root = ProvidedTypeDefinition(asm, ns, "ZipProvider", Some(typeof<obj>))
    let filePathParam = ProvidedStaticParameter("FilePath", typeof<string>)

    let buildTypes (typeName:string) (args:obj[]) =
        let fileName = args.[0] :?> string
        ...

    do root.DefineStaticParameters ([filePathParam], buildTypes)
    do this.AddNamespace(ns, [root])
        
[<TypeProviderAssembly>]
do()
```

Assuming `buildTypes` is complete and working the following user code might be used to use the Type Provider:

```fsharp
type myZip = Xebex.ZipProvider<"myfile.zip">

let file1Crc = myZip.MyFile1.Crc
let file1Size = myZip.MyFile1.Size
```

I always try think how the Type Provider might be used before undertaking work like this, Type Providers are supposed to aid usability not hinder it.  I'm not a fan of big mechanistic design sessions and pencil pushing, I like to get into the field and working things out, that's just my way though.  

## Build it and they will come
Next we need to create types based on the output of SharpCompress.  The property `zipFile.Entries` returns a sequence of `IArchiveEntry` which have properties such as `Size`, `Crc`, `FileName` etc, so we'll use these as we construct the type system.  

One thing to be aware of with SharpCompress is the `Entries` properties returns a flat list of all the files in the archive.  If you have a simple archive with only files at the root level then things are very simple.  Once you move to an archive that has a complex directory hierarchy then things get a little trickier.  One of the reasons is type namespace collisions, if we have file's with the same name but different directories then the type system needs to match this to avoid adding a type with the same name.  It doesn't really make sense to have a flattened list anyway as I was using this provider to quickly peruse zip files from [FSI][4].  

Here's the bulk of `buildTypes`: 

```fsharp
let buildTypes (typeName:string) (args:obj[]) =
    let fileName = args.[0] :?> string
    let zipfile = SharpCompress.Archive.ArchiveFactory.Open(fileName)
    let zipType = ProvidedTypeDefinition(asm, ns, typeName, Some(typeof<obj>))

    ...

    for entry in zipfile.Entries do

        //we need to add types for each directory before adding the zipEntryType to the last occurrence
        let dirs = Path.getAllDirectories entry.FilePath
        let parent = processDirectories dirs zipType

        if entry.IsDirectory then
            parent.AddMembers <| mkProperties entry
        else
            let zipEntry = ProvidedTypeDefinition(safeTypeName entry.FilePath, Some(typeof<obj>))
            zipEntry.AddMembers <| mkProperties entry
            parent.AddMember(zipEntry)

    zipType
```

There are a few parts of code missing, but I'll get to those in a second.  You can see we create a root type to hold the type system that will represent the zip file:  
```fsharp
let zipType = ProvidedTypeDefinition(asm, ns, typeName, Some(typeof<obj>))
```

SharpCompress is then used to open the archive and loop over the entries.  For each file entry found we create a `ProvidedTypeDefinition` and corresponding properties and add it to the parent, but for each directory we only add properties to an existing ProvidedType.  

The important functions missing here are `mkProperties`, `getAllDirectories` and `processDirectories`

### mkProperties
`mkProperties` is the meat and potatoes here:  

```fsharp
let mkProperties (entry:IArchiveEntry) = 
    [yield PP.MkStatic ("FilePath", fun _ -> Expr.Value entry.FilePath)
     if not entry.IsDirectory then yield PP.MkStatic ("Crc", fun _ -> Expr.Value entry.Crc)
     yield PP.MkStatic ("PackedSize", fun _ -> Expr.Value entry.CompressedSize)
     yield PP.MkStatic ("Size", fun _ -> Expr.Value entry.Size)
     yield PP.MkStatic ("CompressionRatio", fun _ -> Expr.Value (float entry.Size / float entry.CompressedSize))
     yield PP.MkStatic ("SpaceSavings", fun _ -> Expr.Value (1.0 - float entry.CompressedSize / float entry.Size))]
```

This function take a `IArchiveEntry` and returns a bunch of `ProvidedProperties`, one for each property we are interested in exposing in our type system.  `PP` is a [Type Abbreviation][6] for `ProvidedProperty`, `MkStatic` is a [Type Extension][5], these will both explained further on.  In this function we are creating a list comprehension with each of the properties we want to represent.  `MkStatic` is just a wrapper around the `ProvidedProperty` constructor, each property has a name, type and the getter function as represented by an expression.  In this instance our expression is just the value of the property in `IArchiveEntry` so we represent this with `Expr.Value entry.x`.  You might of been tempted to write this expression as <@@ entry.x @@> which uses the [Untyped Quotation][7] syntax but this would of resulted in an error from the compiler when in use.  This is to do with type erasure, and the fact that only simple types can be represented as values in the quotation blocks.  There's a [stackoverflow question][8] that covers this too.  The last two properties and not simple properties but calculations, that's one of the beauties of Type Providers, you can easily leverage an existing API and make it more usable for your domain.  

### getAllDirectories
`getAllDirectories` is a module that extends `Path` so that a list of directory elements are returned for a path string.  e.g. "/Users/dave/test" would yield ["Users"; "dave"; "test"].  We use this in `processDirectories` to ensure that each part of the path has a corresponding type stemming from the root.  This ensures the ZipProvider provides the same hierarchy as a file browser.  To be fair I've reinvented the wheel as this functionality is in `Uri.Segments`, but this serves as a *how-to* on extending existing type to bend them to your will!  *(That's my excuse anyway!)*  

```fsharp
module Path =
    let getAllDirectories (path:string) =
        let dname = Path.GetDirectoryName path
        dname.Split ([|Path.DirectorySeparatorChar|], StringSplitOptions.RemoveEmptyEntries)
        |> List.ofArray
```

### processDirectories
`processDirectories` is a recursive function that takes a list of directories and an initial base type and ensures that each directory has been assigned a type and a valid parent.  Once the function has processed the entire path the last `ProvidedTypeDefintion` is returned from the function.  You can see this used in `buildTypes` to either add files or directory properties as explained above.  Sometime recursive functions can take a while to click in you brain, the secret here is in the `acc` or accumulator parameter which is the current parent that's used to add the next type to.  

```fsharp
let directoriesAdded = Dictionary<_,_> ()

let processDirectories directories (root:ProvidedTypeDefinition) =
    let rec loop list (acc:ProvidedTypeDefinition) =
        match list with
        | currentDir :: t ->
            if directoriesAdded.ContainsKey currentDir
            then loop t directoriesAdded.[currentDir]
            else
                //create provided type definition
                let pt = ProvidedTypeDefinition(currentDir, Some(typeof<obj>))
                //add to parent provided type
                acc.AddMember pt
                //add to dictionary
                directoriesAdded.Add (currentDir, pt)
                //recurse
                loop t pt

        | [] -> (*return acc on completion*) acc
    loop directories root
```

There's also `safeTypeName` shown below, essentially this makes sure the type name is just the last segment of the path and that it doesn't have leading or trailing slashes.  

```fsharp
let safeTypeName name =
    //try get just the filename
    let filename = Path.GetFileName(name)
    //if it's empty then it will be a directory
    if String.IsNullOrEmpty filename
    then name.Trim [|Path.DirectorySeparatorChar|]
    else filename
```

## And Another Thing ...

Oh I almost forgot, the type extension and type abbreviation I mentioned above, I used these to make things a little easier:  

```fsharp
type PP = ProvidedProperty

type ProvidedProperty =
    static member MkStatic<'a> (name, getter, ?setter) =
        let pp = PP (name, typeof<'a>, IsStatic = true, GetterCode = getter)
        setter |> Option.iter (fun v -> pp.SetterCode <- v)
        pp
```

I added the `MkStatic<'a>` extension to slim down the code necessary to create a `ProvidedProperty`, without this the creation of a `ProvidedProperty` would be a little longer:  

```fsharp
let pp = ProvidedProperty ("name", typeof<mytype>, IsStatic = true, GetterCode = (fun _ -> Expr.Value 42)) 
```

It's pure laziness, I get sick of typing `typeof<'a>` all the time, and the object initializer property names like `GetterCode = ...`.  The same goes for the type abbreviation.  If I find myself typing a lot of repetitive long type names like `ProvidedProperty` then why not shorten it to PP.  I do this  when working with quotation types too.  

If you wondering about the namespace I use, *Xebec*, its part of a suite of things I've been working on and off for a while involving lots of different things, it just my private codename I use...  

## 99 Ways To Die

OK here's all the code from top to bottom 99 lines. I don't really like to duplicate but after the explanation about it will probably (hopefully) make sense now to read through.  

```fsharp
namespace Xebex.Zip

open System
open System.Collections
open System.Collections.Generic
open System.IO
open System.Reflection
open System.Collections
open Microsoft.FSharp
open Microsoft.FSharp.Core.CompilerServices
open Microsoft.FSharp.Quotations
open ProviderImplementation.ProvidedTypes
open SharpCompress.Archive

module Path =
    let getAllDirectories (path:string) =
        let dname = Path.GetDirectoryName path
        dname.Split ([|Path.DirectorySeparatorChar|], StringSplitOptions.RemoveEmptyEntries)
        |> List.ofArray

type PP = ProvidedProperty

type ProvidedProperty =
    static member MkStatic<'a> (name, getter, ?setter) =
        let pp = PP (name, typeof<'a>, IsStatic = true, GetterCode = getter)
        setter |> Option.iter (fun v -> pp.SetterCode <- v)
        pp

[<TypeProvider>]
type public ZipProvider(cfg : TypeProviderConfig) as this =
    inherit TypeProviderForNamespaces()

    let asm = Assembly.GetExecutingAssembly()
    let ns = "Xebec"
    let root = ProvidedTypeDefinition(asm, ns, "ZipProvider", Some(typeof<obj>))
    let filePathParam = ProvidedStaticParameter("FilePath", typeof<string>)

    let buildTypes (typeName:string) (args:obj[]) =
        let fileName = args.[0] :?> string
        let zipfile = SharpCompress.Archive.ArchiveFactory.Open(fileName)
        let zipType = ProvidedTypeDefinition(asm, ns, typeName, Some(typeof<obj>))

        let directoriesAdded = Dictionary<_,_> ()

        let processDirectories directories (root:ProvidedTypeDefinition) =
            let rec loop list (acc:ProvidedTypeDefinition) =
                match list with
                | currentDir :: t ->
                    if directoriesAdded.ContainsKey currentDir
                    then loop t directoriesAdded.[currentDir]
                    else
                        //create provided type definition
                        let pt = ProvidedTypeDefinition(currentDir, Some(typeof<obj>))
                        //add to parent provided type
                        acc.AddMember pt
                        //add to dictionary
                        directoriesAdded.Add (currentDir, pt)
                        //recurse
                        loop t pt

                | [] -> (*return acc on completion*) acc
            loop directories root

        let safeTypeName name =
            //try get just the filename
            let filename = Path.GetFileName(name)
            //if it's empty then it will be a directory
            if String.IsNullOrEmpty filename
            then name.Trim [|Path.DirectorySeparatorChar|]
            else filename

        let mkProperties (entry:IArchiveEntry) = 
            [yield PP.MkStatic ("FilePath", fun _ -> Expr.Value entry.FilePath)
             if not entry.IsDirectory then yield PP.MkStatic ("Crc", fun _ -> Expr.Value entry.Crc)
             yield PP.MkStatic ("PackedSize", fun _ -> Expr.Value entry.CompressedSize)
             yield PP.MkStatic ("Size", fun _ -> Expr.Value entry.Size)
             yield PP.MkStatic ("CompressionRatio", fun _ -> Expr.Value (float entry.Size / float entry.CompressedSize))
             yield PP.MkStatic ("SpaceSavings", fun _ -> Expr.Value (1.0 - float entry.CompressedSize / float entry.Size))]

        for entry in zipfile.Entries do

            //we need to add types for each directory before adding the zipEntryType to the last occurrence
            let dirs = Path.getAllDirectories entry.FilePath
            let parent = processDirectories dirs zipType

            if entry.IsDirectory then
                parent.AddMembers <| mkProperties entry
            else
                let zipEntry = ProvidedTypeDefinition(safeTypeName entry.FilePath, Some(typeof<obj>))
                zipEntry.AddMembers <| mkProperties entry
                parent.AddMember(zipEntry)

        zipType

    do root.DefineStaticParameters ([filePathParam], buildTypes)
    do this.AddNamespace(ns, [root])
        
[<TypeProviderAssembly>]
do()
```  

So if you made it this far you have seen: Type Providers, recursive functions, list comprehensions, type extensions, type abbreviations, object initialisers, pattern matching, and quotations, quite a few F# features!  

Until next time!

{{< EssentialListening
    "|http://upload.wikimedia.org/wikipedia/en/8/85/Axiscover.jpg|The Jimi Hendrix Experience - Axis: Bold as Love"
    "|http://upload.wikimedia.org/wikipedia/en/b/bf/JimiHendrixValleysOfNeptune.jpg|The Jimi Hendrix Experience - Valleys of Neptune" >}}

[1]: http://www.h2g2.com
[2]: http://msdn.microsoft.com/en-us/library/hh156509.aspx
[3]: https://sharpcompress.codeplex.com
[4]: http://msdn.microsoft.com/en-us/library/dd233175.aspx
[5]: http://msdn.microsoft.com/en-us/library/dd233211.aspx
[6]: http://msdn.microsoft.com/en-us/library/dd233246.aspx
[7]: http://msdn.microsoft.com/en-us/library/dd233212.aspx
[8]: http://stackoverflow.com/questions/10161437/type-provider-providing-me-with-an-unsuported-constant-type-system-double-er

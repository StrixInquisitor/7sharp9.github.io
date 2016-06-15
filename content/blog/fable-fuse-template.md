+++
author = "7sharp9"
categories = ["programming"]
tags = ["F#", "Fuse", "Fable", "Transpiling", "JavaScript"]
date = "2016-06-07T13:20:49+01:00"
description = "A fable-fuse how to"
featured = "fable-fuse.png"
featuredalt = "fable-fuse.png"
featuredpath = "/img/fable-fuse"
linktitle = ""
title = "Creating fuse applications with fable"
type = "post"
images = ["http://7sharpnine.com/img/fable-fuse/fable-fuse-tc.png?6"]
+++
So as promised here's a guide to creating your first **fable |> fuse** application<!--more-->

I've made a template available so this can be tried out quickly and easily, I'll run through the 
requirements and describe whats in the template.  

# Requirements
First of all here are the requirements.  

## Yeoman
As [fable][1] uses [npm modules][12] for dependencies **fable |> fuse** template will also be based 
on them.  I have used [Yeoman][4] as it's very flexible and works nicely.  

So what you need to get Yeoman is:

```bash
npm install -g yo
```
And then to install the **fable |> fuse** template for Yeoman:

```bash
npm install -g generator-fable-fuse
```
Nice and easy.

## Fable
Installing Fable is covered in the [Fable documentation][4], but it's essentially just another npm install:

```
npm install -g fable-compiler
```

## Fuse
Fuse can be downloaded at their site [here][5]

That's pretty much all the requirements, let's try this out!  
- - -
# Creating a **fable |> fuse** application?
OK, so with everything installed how do you get going?

This is really easy:

```bash
mkdir fable-fuse-test
cd fable-fuse-test
yo fable-fuse
```

And then follow the prompts, here's a [asciinema][3] session showing the process:

{{< figure src="https://asciinema.org/a/9eqmwurfreaspzsmrm0f5ezc9.png" link="https://asciinema.org/a/9eqmwurfreaspzsmrm0f5ezc9?size=medium&speed=1.75" class="8u" >}}

Now that the template has been created let's have a look at whats inside.  
- - -
# Project Structure

The structure of a **fable |> fuse** application is as follows *(Using a project name of test)*:

```
├── App
│   ├── MainView.ux
│   └── test.unoproj
├── build.bat
├── build.sh
├── node_modules
│   ├── fable-core
│   ├── fable-fuse
│   └── fable-import-fetch
├── package.json
└── src
    ├── fableconfig.json
    ├── test.fs
    └── test.fsproj
```
Lets go through the root files first

## package.json  
This file has the dependencies for `fable |> fuse`.  Currently these are: 

* **fable-core** : This has the main definitions for Fable.  
* **fable-fuse** : This has the bindings for the Fuse JavaScript API's.  
* **fable-import-fetch** : This has the F# bindings for JavaScript Fetch API.    

## build.sh / build.bat  
These two files contain the script to transpile the F# source into JavaScript.  So upon typing `./build.sh` the F# files 
will be transpiled into JavaScript and placed into the `App/js` folder.  In addition [Fable][1] will continue to watch the 
F# files and transpile the files if they change so you get realtime updating of the [fuse][2] application.

Now the directories:

## App 
The `App` directory has all the necessary files that [Fuse][2] requires to build

### test.unoproj
This is the project file for Fuse, it has settings for the different platforms and controls which gets embedded in the application.  

### MainView.ux
This is the main view markup file for the user interface.  

## src
The `src` folder has all the F# source files that [Fable][1] transpiles into JavaScript.

### fableconfig.json
This is the configuration file for Fable which allows you to run scrip before or after compilation and set various defaults.  

### test.fs
This is the main source file for F# and is the same as the one you saw in the previous post:

```fsharp
namespace App
open Fable.Core
open Fuse
open Fable.Import
open Fable.Import.Fetch

module test =
    let data = Observable.create()

    promise {
        let! req = GlobalFetch.fetch (Url "http://az664292.vo.msecnd.net/files/ZjPdBhWNdPRMI4qK-colors.json")
        let! json = req.json ()
        do (data.value <- json) } |> ignore
```

This is fairly easy to follow, the `promise` block is a custom [computation expression][13] that allows each successful JavaScript 
promise to execute before the next promise is ran.  In this example `GlobalFetch.fetch` and `req.json()` both return JavaScript 
promises. The promise block runs the `GlobalFetch.fetch` function and if it *succeeds* it runs the `req.json()` 
function.  If that too is successful then the observable value `data` is updated to the resulting json data.  

## node-modules  
These are our dependencies, there's fable-core which is required by [Fable][1], also included are [fable-fuse][8] which 
are the F# bindings to the Fuse JavaScript libraries, and [fable-import-fetch][7] which is the F# bindings for the 
[Fetch JavaScript API][9].  
- - -
# Running
To get a **fable |> fuse** application running all you have to do is run the build script `./build` which transpiles the 
F# files into JavaScript and then watches for any updates to the F# files.  Running a Fuse application is also really easy, 
you can do this from within Atom via the [plugin][10], or sublime via that [plugin][11], or simply just run 
`fuse preview ./App/` from the project root.  
- - -
# Any Problems?
If you have any problems with the Yeoman generator for **fable |> fuse** then please log an issue on its 
GitHub repo: [generator-fable-fuse][14].  If you have any issues with the fable-fuse module itself then please got an issue 
on its GitHub repo: [fable-fuse][15].  

If you have an improvements or suggestions then a PR is very welcome too!  
- - -
# Whats next?
If there is enough interest around using **fable |> fuse** I'll port some of the more intricate samples from the 
[Fuse examples][6] over to **fable |> fuse** and also create a GitHub site with all the content relating to it.  

Let me know what you think!!

Until next time!
- - -

[1]: http://fsprojects.github.io/Fable/
[2]: https://www.fusetools.com/
[3]: https://asciinema.org
[4]: http://fsprojects.github.io/Fable/docs.html
[5]: https://www.fusetools.com/downloads
[6]: https://www.fusetools.com/examples
[7]: https://www.npmjs.com/package/fable-import-fetch
[8]: https://www.npmjs.com/package/fable-fuse
[9]: https://developer.mozilla.org/en/docs/Web/API/Fetch_API
[10]: https://atom.io/packages/fuse
[11]: https://github.com/fusetools/Fuse.SublimePlugin
[12]: https://docs.npmjs.com/getting-started/what-is-npm
[13]: https://msdn.microsoft.com/visualfsharpdocs/conceptual/computation-expressions-%5Bfsharp%5D
[14]: https://github.com/7sharp9/generator-fable-fuse
[15]: https://github.com/7sharp9/fable-fuse

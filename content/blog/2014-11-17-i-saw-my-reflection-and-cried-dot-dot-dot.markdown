---
author: 7sharp9
layout: post
title: "I saw my reflection and cried ..."
date: 2014-11-16
comments: true
categories: [programming]
tags: [TypeProviders,F#,reflection]
description: ""
type: post
---

While I was visiting Boston earlier in the year I had the misfortune of kicking myself in the teeth with reflection.  It's something all programmers inevitably go through with reflection API's as they are inherently untyped, a simple typo can leave you tearing out your hair or punching through your monitor!  Yeah there's things the horizon that will help namely the [nameof][1] expression in C#6 which should help in some areas, that's if your willing to pay the price of using C#, but I wont go into that here :-).  In F# we can leverage Type Providers fairly easily to wrap API usages in cases that we are interested in, or even create a general usage with a little more effort.  
<!-- more -->

### Using the Type Provider
In usage it will look like this vs the usual reflection API:

```fsharp

//traditional reflection using untyped method
let tt = typeof<DateTime>
let meth = tt.GetMethod("Add")
let result = meth.Invoke(DateTime.Now, [|TimeSpan.FromDays(1.)|])

//using the type provider to provide a little safety net
type rt = TypedReflection.Reflection< "System.DateTime", "AddSeconds">
let result = rt.AddSeconds(DateTime.Now, 1.)
```

If you make a mistake the compiler will tell you and you will be forces to fix the typo or add namespace prefixes etc.  You also get intellisense.autocompletion on usage and you can give actual parameters rather than arrays of loose objects etc.  

### Code Dump
First of all I'm just going to leave the code here, and then talk through it below:

```fsharp
[<TypeProvider>]
type public ReflectionTypeProvider(config : TypeProviderConfig) as this = 
    inherit TypeProviderForNamespaces()

    let assembly = Assembly.GetExecutingAssembly()
    let nameSpace = this.GetType().Namespace
    let providerType =
        ProvidedTypeDefinition(assembly, nameSpace, "Reflection", Some typeof<obj>, 
                               IsErased = true, HideObjectMethods = true)

    let buildReflection typeName (parameters : obj[]) =  
        let reflectionType = string parameters.[0]
        let methodName = string parameters.[1]

        let theType = Type.GetType(reflectionType, true)
        let meth = theType.GetMethod(methodName)
        if meth = null then failwith "No such method!"

        let wrapper = ProvidedTypeDefinition(assembly, nameSpace, typeName, Some (typeof<obj>),
                                             HideObjectMethods = true )

        let parameterInfoToProvidedParameter (meth:MethodInfo) =
            let pi = meth.GetParameters()

            let instance = ProvidedParameter("instance", meth.ReflectedType)
            let parameters =
                pi
                |> Seq.map (fun p -> ProvidedParameter(p.Name, p.ParameterType) )
                |> Seq.toList
            instance :: parameters
            
        let reflectionWrapper =
            ProvidedMethod (meth.Name, parameterInfoToProvidedParameter meth, meth.ReturnType,
                            IsStaticMethod = true,
                            InvokeCode = function
                                         | instance :: parameters ->
                                             try Expr.Call (instance, meth, parameters)
                                             with exn -> failwith "Error creating Invoke code."
                                         | _ -> failwith "Error: unexpected number of parameters" )
        wrapper.AddMember reflectionWrapper
        wrapper

    do 
        providerType.DefineStaticParameters
            ([ ProvidedStaticParameter("Type", typeof<string>)
               ProvidedStaticParameter("Method", typeof<string>) ], 
             buildReflection)

        this.AddNamespace (nameSpace, [ providerType ])

[<assembly:TypeProviderAssembly>] 
do()
```

### Skeleton code
Reading from the bottom up you can see the parameters that our Type Provider accepts are `Type` and `Method`, those a pretty self explanatory.  You should also notice other boiler plate Type Provider code if you read my last [ZipProvider post][2].  The important part here is the `buildReflection` function.  

### buildReflection
First of all on lines `12/13` we scrape of the configuration parameters `theType` and `meth`, we then do a quick check to ensure the type and method actually exist, if they don't we raise an error on line `17` so the use can correct the code.  

Next we create a variable named wrapper which *wraps* round the reflection API by creating a `ProvidedTypeDefinition` on line `19`.  We now have two methods which we use to create our safe API, `parameterInfoToProvidedParameter` and `reflectionWrapper`.  

### parameterInfoToProvidedParameter
The purpose of this is a mapping function from the reflection API's untyped abstract form to our typed form that we use in the construction of the Provided methods.  Essentially this is pretty simple, we get the parameters for the `MethodInfo` which we are wrapping on line `23`.  The first parameter will be the instance of the reflected method will be working on, and the rest of the parameters will be those of the reflected method.  To add those we loop over the parameters from the `MethodInfo` and map then to `ProvidedProperties` by using the `Name` and `ParameterType` properties.  

*(Thinking about this we could do it slightly differently by adding a `ProvidedConstructor` which could take the initial instance, this could be added fairly easily if we really needed it.  )*  

### reflectionWrapper
The reflectionWrapper is where the magic happens, we create a `ProvidedMethod` using the `MethodInfo`'s name', we add the parameters by using the `parameterInfoToProvidedParameter` function, and we also add the return type by using the `MethodInfo`'s `ReturnType` parameter'.  We can also take advantage of [object initializers][5] here to set `IsStaticMethod` to true, and to add in the invoke code.

The invoke code uses the `function` keyword which is really just a pattern match expression using only a single argument, here we use pattern matching on a list to extract the `head|tail` arguments.  If you remember the `parameterInfoToProvidedParameter` function then you will know that it returns a list `instance :: parameters`.  We can now use the [Quotations `Expr`][3] type with the [`Call`][4] function and pass in our instance and parameters in directly (instance is the reflected methods instance type, `meth` is the `MethodInfo` we will be calling, parameters are the parameters the `MethodInfo` requires.  

###  Wrapping up
Finally we just add the `ProvidedMethod` *`reflectionWrapper`* to the `ProvidedType` *`wrapper`*

This is a fairly simple implementation but it could be **beefed up** quite easily into something a little more elaborate without too much trouble.  If you use your imagination then there are numerous possibilities with Type Providers!

Reminds me of an old proverb:

```
If you have a problem ...  
if no one else can help ...  
and if you cant find an existing one ...  
maybe you can build ...  
a Type Provider.
```

:-)

Until next time!

* * *
#### Essential listening:  
*   Alice In Chains - Dirt
*   Alice In Chains - Alice In Chains  

{{< img-fit
    2u "http://upload.wikimedia.org/wikipedia/en/b/ba/Dirt.jpg" "Alice In Chains - Dirt"
    2u "http://upload.wikimedia.org/wikipedia/en/2/24/Alice_in_Chains_%28album%29.jpg" "Alice In Chains - Alice In Chains" "" "" >}}

[1]: http://msdn.microsoft.com/en-us/magazine/dn802602.aspx
[2]: http://7sharpnine.com/posts/flux-compression-redux/
[3]: http://msdn.microsoft.com/en-gb/library/ee370577.aspx
[4]: http://msdn.microsoft.com/en-us/library/ee370395.aspx
[5]: http://msdn.microsoft.com/en-us/library/dd233192.aspx#sectionToggle4

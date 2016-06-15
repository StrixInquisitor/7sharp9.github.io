---
author: 7sharp9
layout: post
title: "Anything you can do ..."
date: 2014-06-01
comments: true
categories: [programming]
tags: [F#, Xamarin, C#, ObjC]
description: ""
type: post
---
For any of you that are aware of the newly updated Xamarin Web site, you may have seen the [following][1]:

>Objective-C was ahead of its time 30 years ago.
C# is ahead of its time today.
Anything you can do in Objective-C or Java, you can do in C# with Xamarin—usually more succinctly and with fewer bugs.

What is also true is that F# is way ahead of its time, and you can produce even more succinct code with even fewer bugs than C#!  
<!-- more -->
Take the code snippets from that page.

First up the Objective C version:
```
@interface Person : NSObject
@property (strong, nonatomic) NSString *name;
@end

@implementation Person
- (id)initWithName:(NSString *)name {
    self = [super init];
    if (self) {
        self.name = name;
    }
    return self;
}

+ (NSArray *)getNames
{
    NSArray *people = @[
      [[Person alloc] initWithName:@"David"],
      [[Person alloc] initWithName:@"Vinicius"],
      [[Person alloc] initWithName:@"Serena"],
    ];
    NSMutableArray *names = [NSMutableArray array];
    for (Person *person in people) {
        [names addObject:person.name];
    }
    return names;
}
@end
```

Heres the C# version:
```csharp
class Person : NSObject {
  public string Name { get; set; }
  
  public static string[] GetNames() {
    var people = new[] {
        new Person { Name="David" },
        new Person { Name="Vinicius" },
        new Person { Name="Serena" },
    };
    return people.Select(person => person.Name).ToArray();
  }
}
```

And finally the F# version:
```fsharp
type Person() =
   inherit NSObject()
   member val Name = "" with get, set
   static member GetNames() =
      [| new Person(Name="David")
         new Person(Name="Vinicius")
         new Person(Name="Serena") |]
      |> Array.map(fun person -> person.Name)
```

You can see the F# version is doing exactly the same, although we are using the `map` function from the [`Array` module][4] rather than the [Linq][3] `Select` extension method.  

Its not all about the lines of code though, using F# gives you all many advantages:

*   Using the type system to make sure the code is behaving how you expect before you even compile.  
*   [Pattern matching][5] in F# is amazing!  It can vastly simplify complex control logic, add Active patterns to that and you are ready to take on the world!  
*   Problems are approached from a functional perspective which often leads to succinct functions that are easy to reason about, test, and compose.  
*   F# emphasizes immutability and functional composition rather than inheritance, again this boils down to simplicity.  
*   Features like [Type providers][2] can vastly simplify how you deal with data within your application, making access to data really easy and intuitive.  

Theres are many areas that F# can really help productivity during development.  I hope to write a few more short posts to really bring attention to these.  I use F# all the time and often forget how awesome it is until I go back to another language thats missing those features.  

Until next time!  

* * *
#### Essential listening:
*   Sacred Reich - Independent  
*   Xentrix - For Whose Advantage   

{{< img-fit	
    2u "http://upload.wikimedia.org/wikipedia/en/a/a6/Sr_independent.jpg""Sacred Reich - Independent"
    2u "http://upload.wikimedia.org/wikipedia/en/4/46/For_Whose_Advantage.jpg" "Xentrix - For Whose Advantage" "" "" >}}

 [1]: https://xamarin.com/platform
 [2]: http://msdn.microsoft.com/en-gb/library/hh156509.aspx
 [3]: http://msdn.microsoft.com/en-us/library/bb397926.aspx
 [4]: http://msdn.microsoft.com/en-us/library/ee370273.aspx
 [5]: http://msdn.microsoft.com/en-gb/library/dd547125.aspx
 [6]: http://msdn.microsoft.com/en-us/library/dd233248.aspx
 
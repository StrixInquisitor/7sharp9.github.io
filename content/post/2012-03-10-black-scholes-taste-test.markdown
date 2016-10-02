---
layout: post
title: "Black-Scholes Taste Test"
slug: Black-Scholes-Taste-Test
date: 2012-03-11
comments: true
categories: [programming]
tags: [FSharp, CSharp, Finance, Performance, Compilation]
description: ""
type: post
---
In this edition we are going to be doing a taste test, C# vs F#.  Oh yeah, if you quickly glanced at the title you may 
have thought this was a recipe for black scones, as interesting and tasty as that may be, unfortunately its going 
to be finance related.    

I recently presented a paper on the benefits of F#, part of this was a comparison of the famous 
[Black-Scholes](http://en.wikipedia.org/wiki/Black-Scholes) equation in both C# and F#.  I was mainly going to be 
looking at code succinctness and the inherent suitability of the language for calculation based work, but there ended 
up being more to it than that.

First of all I quickly set up a test rig to run 50 million iterations of the algorithm to see if there were any difference 
in the processing speed.  I want expecting any major differences at this point but here's what I got:

C# results for 50 million iterations
{{< figure src="https://lh6.googleusercontent.com/-cEzGoE_P2cE/T1vf_SxtGfI/AAAAAAAABRE/RE4ReRLAhu8/s531/csbs.png" >}}

F# results for 50 million iterations
{{< figure src="https://lh3.googleusercontent.com/-PLdltL0YiIs/T1vf_Wo2ZrI/AAAAAAAABRI/WijGdNaOnK4/s531/fsbs.png" >}}

I think you will agree that's quite a difference, lets have a look at the code to see what's going on.

## C# Implementation
```
public class Options
{
    public enum Style
    {
        Call,
        Put
    }

    public static double BlackScholes(Style callPut, double s, double x, double t, double r, double v)
    {
        double result = 0.0;
        var d1 = (Math.Log(s / x) + (r + v * v / 2.0) * t) / (v * Math.Sqrt(t));
        var d2 = d1 - v * Math.Sqrt(t);
        switch (callPut)
        {
            case Style.Call:
                result = s * Cnd(d1) -x * Math.Exp(-r * t) * Cnd(d2);
                break;
            case Style.Put:
                result = x * Math.Exp(-r * t) * Cnd(-d2) -s * Cnd(-d1);
                break;
        }
        return result;
    }

    private static double Cnd(double x)
    {
        const double a1 = 0.31938153;
        const double a2 = -0.356563782;
        const double a3 = 1.781477937;
        const double a4 = -1.821255978;
        const double a5 = 1.330274429;
        var l = Math.Abs(x);
        var k = 1.0 / (1.0 + 0.2316419 * l);
        var w = 1.0 - 1.0 / Math.Sqrt(2 * Math.PI) * 
            Math.Exp(-l * l / 2.0) * (a1 * k + a2 * k * k + a3 * 
                Math.Pow(k, 3) + a4 * Math.Pow(k, 4) + a5 * Math.Pow(k, 5));
        if (x < 0)
        {
            return 1.0 - w;
        }
        return w;
    }
}
```
## F# Implementation
```fsharp
module options
open System

type Style = Call | Put
  
let cnd x =
   let a1 = 0.31938153
   let a2 = -0.356563782
   let a3 = 1.781477937
   let a4 = -1.821255978
   let a5 = 1.330274429
   let l  = abs x
   let k  = 1.0 / (1.0 + 0.2316419 * l)
   let w  = (1.0 - 1.0 / sqrt(2.0 * Math.PI) * 
                exp(-l * l / 2.0) * (a1 * k + a2 * k * k + a3 * 
                    (pown k 3) + a4 * (pown k 4) + a5 * (pown k 5)))
   if x < 0.0 then 1.0 - w
   else w

let blackscholes style s x t r v =
    let d1 = (log(s / x) + (r + v * v / 2.0) * t) / (v * sqrt(t))
    let d2 = d1 - v * sqrt(t)
    match style with
    | Call -> s * cnd(d1) -x * exp(-r * t) * cnd(d2)
    | Put -> x * exp(-r * t) * cnd(-d2) -s * cnd(-d1)
```
## Differences
The most significant differences when the code is compiled comes down to a few areas.  

### The BlackScholes function
The first thing to note is the code size and number of local variables:
```
// Code size       122 (0x7a)
.maxstack  6
.locals init ([0] float64 d1,
         [1] float64 d2)
```
```
// Code size       164 (0xa4)
.maxstack  4
.locals init ([0] float64 d1,
         [1] float64 d2,
         [2] float64 result,
         [3] valuetype CsBs.Options/Style CS$0$0000)
```
The initial arguments that are loaded in the F# implementation is done in fewer IL op codes then C#.  
```
IL_0001:  ldarg.1
IL_0002:  ldarg.2
IL_0003:  div
IL_0004:  call       float64 [mscorlib]System.Math::Log(float64)
```
```
IL_0000:  ldc.r8     0.0
IL_0009:  stloc.0
IL_000a:  ldc.r8     0.0
IL_0013:  stloc.1
IL_0014:  ldc.r8     0.0
IL_001d:  stloc.2
IL_001e:  ldarg.1
IL_001f:  ldarg.2
IL_0020:  div
IL_0021:  call       float64 [mscorlib]System.Math::Log(float64)
```
You can see in the C# code is intialising the local variable to 0.0 by pushing them to the stack 
`ldc.r8` then storing them `stloc.0`.

The pattern matching in the F# code results in a call to get the style `options/Style::get_Tag()` 
and then a branch if not equal opcode `bne.un.s` which causes a jump to `IL_005d`
```
IL_0036:  call       instance int32 options/Style::get_Tag()```
IL_003b:  ldc.i4.1
IL_003c:  bne.un.s   IL_005d
```
The C# version loads the local variable for the `Style` `IL_0053:  stloc.3` and then uses the switch 
opcode to jump table to jump to either position `IL_0064` or `IL_0083`.
```
IL_0053:  stloc.3
IL_0054:  ldloc.3
IL_0055:  switch     ( 
                      IL_0064,
                      IL_0083)
IL_0062:  br.s       IL_00a2
```
These are negligible, I'm mealy pointing out the differences in compilation between the two languages.  
The F# compiler is more stringent when compiling the code.

### The Cnd function
The Cnd function or [cumulative normal distribution](http://en.wikipedia.org/wiki/Normal_distribution) 
is where the performance differences occur.

Again at initialization you can see the C# version is larger by 41.  
```
// Code size       213 (0xd5)
.maxstack  8
.locals init ([0] float64 l,
         [1] float64 k,
         [2] float64 w)
```
```
// Code size       254 (0xfe)
.maxstack  6
.locals init ([0] float64 l,
         [1] float64 k,
         [2] float64 w)
```
The C# version initialises all the local variables to 0.0.
```
IL_0000:  ldc.r8     0.0
IL_0009:  stloc.0
IL_000a:  ldc.r8     0.0
IL_0013:  stloc.1
IL_0014:  ldc.r8     0.0
IL_001d:  stloc.2
```
Interestingly the C# compiler optimises out the call to `Math.PI * 2` but the F# compiler doesn't.

```
IL_003a:  ldc.r8     2.
IL_0043:  ldc.r8     3.1415926535897931
IL_004c:  mul
```
```
IL_0057:  ldc.r8     6.2831853071795862
```
From here everything is identical until we get to the power operator section (`Math.Pow` in the C# version and `pown` in F#).

```
IL_0089:  ldloc.1
IL_008a:  ldc.i4.3
IL_008b:  call       float64 [FSharp.Core]Microsoft.FSharp.Core.Operators/OperatorIntrinsics::PowDouble(float64, 
                                                                                                        int32)
``` 
In the F# code we are using the `pown` function which calculates the power to an integer.  This is shown in the 
call to `OperatorIntrinsics::PowDouble` which uses the value in `IL_0089:  ldloc.1` and also loads the 
integer 3 with `IL_008a:  ldc.i4.3`.  
```   
IL_009c:  ldloc.1
IL_009d:  ldc.r8     3.
IL_00a6:  call       float64 [mscorlib]System.Math::Pow(float64,
                                                        float64)
```
The C# code is using the standard Math.Pow operator which operates on two float64 numbers.  The value of 3 is 
implicitly converted into a `float64` during compilation `IL_009d:  ldc.r8     3.`.  

The final difference is at the end of the function.

```
IL_00b8:  stloc.2
IL_00b9:  ldarg.0
IL_00ba:  ldc.r8     0.0
IL_00c3:  clt
IL_00c5:  brfalse.s  IL_00d3
IL_00c7:  ldc.r8     1.
IL_00d0:  ldloc.2
IL_00d1:  sub
IL_00d2:  ret
IL_00d3:  ldloc.2
IL_00d4:  ret
```
The F# version uses the `clt` opcode.  This pushes 1 if value one on the stack is less than value two otherwise 
it pushes 0.  There is then a `brfalse.s` which jumps to location `IL_00d3` if the first value on the stack is 
less than or equal to the second value.

```
IL_00b8:  stloc.2
IL_00b9:  ldarg.0
IL_00e5:  ldc.r8     0.0
IL_00ee:  bge.un.s   IL_00fc
IL_00f0:  ldc.r8     1.
IL_00f9:  ldloc.2
IL_00fa:  sub
IL_00fb:  ret
IL_00fc:  ldloc.2
IL_00fd:  ret
```
The C# version uses the `bge.un.s` to jump to location `IL_00fc` if the first value on the stack is greater than 
the second.  This is negligible in normal runtime but it is interesting to note the difference between the two.  

## Conclusion
Wow, there was a lot of IL to get through, I hope you stayed with me!

Although the difference in some areas are negligible, every little counts.  The implicit conversion of an integer 
field to a `float64` hides the fact that we were using an optimized integer power function in F#, that's performance 
increase of 168%!  Some other side effects of implicit conversion can also lead to subtle bugs due to truncation 
and overflow.  The other benefits are the compiled code uses less instructions and the source code only uses 25 
lines compared to 44 in C#.

Until next time!
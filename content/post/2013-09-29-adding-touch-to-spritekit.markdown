---
author: 7sharp9
layout: post
title: "Adding Touch To SpriteKit"
date: 2013-09-29
comments: true
categories: [programming]
tags: [FSharp, Games, SpriteKit, iOS, Xamarin]
description: ""
type: post
---
Only a quick post this week.  Last time we looked at [SpriteKit][3] and how to add some particle emmiters to simulate a star-field and exhaust on a spaceship, this time lets look at adding some touch based input to move the spaceship around.  <!-- more -->

The first thing we need to do is add a type of gesture recogniser, there are various built in [gestures][1]: 

   * UITapGestureRecognizer
   * UIPinchGestureRecognizer
   * UIRotationGestureRecognizer
   * UISwipeGestureRecognizer
   * UIPanGestureRecognizer
   * UIScreenEdgePanGestureRecognizer
   * UILongPressGestureRecognizer

From the names above it's pretty easy to get a feel for how they should be used, you can create your own subclass of `UIGestureRecognizer` if you need a custom one.  

Gesture recognizers come in two types continuous and discrete.  A discrete gesture is single action like tap or double tap and results in a single action been sent.  A continuous gesture is like pan, swipe, or rotate which is interpreted as a series of messages being sent.    

For our purposes we are going to be using the `UIPanGestureRecognizer` which is a continuous gesture.  What we need to do is create a function that sets up the `UIPanGestureRecognizer` ready for us to use.  We do that by creating an instance of the `UIPanGestureRecognizer` and add it to our view:  

```fsharp
let setupGestures() =
    use panRecogniser = new UIPanGestureRecognizer(x, MonoTouch.ObjCRuntime.Selector("PanSelector"))
    x.View.AddGestureRecognizer(panRecogniser)
``` 

Here we are also using a selector, which means we can use an attribute like `[<Export("PanSelector")>]` to define the function that will be used as the callback.  Lets define that function now:  

```fsharp
let OnLabelPan( sender: UIGestureRecognizer) =
    match sender with
    | :? UIPanGestureRecognizer as pan ->
        match pan.State with
        | UIGestureRecognizerState.Changed ->
            let movement = pan.TranslationInView(x.View)
            let move = SKAction.MoveBy(movement.X * 1.75f, -movement.Y * 1.75f, 0.05)
            let ship = scene.GetChildNode("Ship")
            ship.RunAction(move)
            pan.SetTranslation(PointF.Empty, x.View)
        | _ -> ()
    | _ -> ()
``` 

First of all we use pattern matching to do a type match `| :? UIPanGestureRecognizer as pan ->`.  This ensures we are dealing with the `UIPanGestureRecognizer` type.  We might of applied multiple gesture recognisers to the view like swipe and rotate and had this function deal with all of them, we can handle this nicely with the type match.  

We can now use a pattern match on the state of the gesture recogniser to react to just the changed event `UIGestureRecognizerState.Changed`.  

As mentioned previously the pan gesture is continuous and will send a changed action whenever the finger moves on the screen, this gives us a chance to retrieve the current translation of the pan in the current view.  We do this by calling `pan.TranslationInView(x.View)`.  We can now apply a movement to our spaceship sprite by creating an action using `SKAction.MoveBy`.   We multiply the translation retrieved by 1.75 to allow for the initial distance the the pan gesture moves before triggering.  We also invert the Y axis so that the spaceship sprite moves in the correct Y direction.  The final parameter is the time the action runs for, we use a really small time of 0.05 (50ms).  This stops the spaceship sprite from moving like an ordinary mouse pointer, just enough inertia to make it feel smooth.  

To apply the action to the spaceship all we need to do is retrieve it from the scene using `scene.GetChildNode` and call the `RunAction` function passing in the action we just created.  

Finally we set the pan translation back to zero using: `pan.SetTranslation(PointF.Empty, x.View)`, this ensures that the spaceship only moves by last changed translation action.  Failure to reset the translation would result in the spaceship having too much inertia from the previous actions making it very difficult to control.  

We could also use another overload of `UIPanGestureRecognizer` which takes an `Action<UIPanGestureRecognizer>`, we can pass this in as a lambda function:    

```fsharp
let setupGestures() =
    use panRecogniser = 
        new UIPanGestureRecognizer
            (fun (pan:UIPanGestureRecognizer) -> 
                match pan.State with
                | UIGestureRecognizerState.Changed ->
                    let movement = pan.TranslationInView(x.View)
                    let move = SKAction.MoveBy(movement.X * 1.75f, -movement.Y * 1.75f, 0.05)
                    let ship = scene.GetChildNode("Ship")
                    ship.RunAction(move)
                    pan.SetTranslation(PointF.Empty, x.View)
                | _ -> ())
``` 

I think the attribute based version is a little cleaner as you can move the callback functionality away from the definition.  To be honest I don't mind either way, although if the lambda definition gets too big you will definitely be better off with the former.  

Finally we need to plug in the `setupGestures` function, we do that by calling it at the end of `ViewDidLoad`:  

```fsharp
override x.ViewDidLoad () =
    base.ViewDidLoad()
    setupScene()
    setupGestures()
```

Here's a quick YouTube video so you can see this in action:   

{{< youtube j5JK5zWLdK0 >}}

If you want to check out the project then you can find it in my [GitHub repo ][2].

That's all for now, see you next time!

{{< EssentialListening
    "|http://upload.wikimedia.org/wikipedia/en/e/e5/In_Utero_%28Nirvana%29_album_cover.jpg|Nirvana - In Utero"
    "|http://upload.wikimedia.org/wikipedia/en/8/85/Axiscover.jpg|Jimi Hendrix Experience - Axis: Bold as Love" >}}


[1]: https://developer.apple.com/library/ios/documentation/uikit/reference/UIGestureRecognizer_Class/Reference/Reference.html#//apple_ref/occ/cl/UIGestureRecognizer
[2]: https://github.com/7sharp9/SpriteKit-Fsharp-Samples
[3]: https://developer.apple.com/library/ios/documentation/GraphicsAnimation/Conceptual/SpriteKit_PG/Introduction/Introduction.html
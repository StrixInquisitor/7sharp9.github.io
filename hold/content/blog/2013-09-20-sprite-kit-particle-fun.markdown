---
author: 7sharp9
layout: post
title: "Spritekit particle fun"
date: 2013-09-20
comments: true
categories: [Programming]
tags: [F#, Games, SpriteKit, iOS, Xamarin]
---
I have been meaning to write this post for quite a while now.  Since the first announcement of the iOS7 beta I immediately saw the list of new API's and **SpriteKit** caught my eye straight away.  I only managed to get time to briefly look over the API and saw that is wasn't the usual trashy API with a million method overloads, internal mutation sucker punch type thing.  It seems to be very declarative and intuitive, which makes for a nice change.  <!-- more -->

First of all lots of kudos to [Xamarin][5] for getting Xamarin iOS 7 out so swiftly, you can read about some of the new features [here][6].

##SpriteKit
So what is SpriteKit?  

SpriteKit, as you might have guessed, is a games oriented API aimed at getting you quickly up and running with 2D sprites so that you can spend more time building your games rather than mucking about with the low level stuff.  Lets have a really quick tour of what's in there, I don't want to spend long on this as you can read the [programming guide][1] on the Apple site for further details.  

###Scenes
SpriteKit represents the different parts of your games with a scenes.  A scene could be the title screen or the levels in the game.  A scene is really just a just a collection of nodes which represents all of the objects currently visible.  There are several different types of node that can be used in the scene.  

###Nodes

Several different type of node are available to use, here they are:  

*   SKVideoNode - Allows videos to be embedded into the scene.
*   SKCropNode - A crop node allows you to mask of different areas of the viewing area.
*   SKEffectNode - The effects node allows its children to be rendered into a private frame buffer where [Core Image][8] effects can be applied before being blended back into the main scene.
*   SKEmitterNode - Allows particles to be placed into a scene.
*   SKLabelNode - Allows for arbitrary text to be places into the scene.
*   SKShapeNode - Allows path based shapes to be draw in the scene.
*   SKSpriteNode - This is your standard textured image which can be colour blended, scaled, rotated etc.
*   SKNode - This is the base node type from which all the others are derived.

###Transitions
Transitions allow you to move from one scene to another, allowing for a animated effect to be applied during the transition.    

###Actions
Actions allow you to declarative apply an action to a node.  For example you could write something like this:  
{% codeblock lang:objc %}
SKAction *moveUp =     [SKAction moveByX:0 y:100.0 duration:1.0];
SKAction *zoom =       [SKAction scaleTo:2.0 duration:0.25];
SKAction *wait =       [SKAction waitForDuration: 0.5];
SKAction *fadeAway =   [SKAction fadeOutWithDuration:0.25];
SKAction *removeNode = [SKAction removeFromParent];
 
SKAction *sequence = [SKAction sequence:@[moveUp, zoom, wait, fadeAway, removeNode]];
[node runAction: sequence];
{% endcodeblock %}

This creates a sequence of actions: moveUp, zoom, wait, fadeAway, removeNode.  The actions are reusable and stateless so they can be applied to any nodes in the scene.  If we didn't want the actions to be applied in a sequence we could use a group, which applies the actions in parallel.  

Don't worry if I've scared you with Objective-C there will be none of that when we get into writing a little demo in a moment.  

###Physics
The physics part of SpriteKit can be really fun to play with, its fairly easy to fill a screen full of cubes and bash them about watching the gravity and collision effects.  The physics engine looks like its based on [Box2D][2] and involves adding approximate shapes for your game objects and then adding a bunch of physical properties like mass, friction, linear damping, restitution etc.  

- - -
###First Steps
That was a really quick whistle stop tour just to give you a flavour of what's in there.  For this post we are going to look at the `SKEmitterNode` and see what we can do.  

The first thing to do is set up the skeleton, not an actual skeleton mind, just the skeleton of the demo.  

Create a new F# iOS Single View Application.  SpriteKit uses a subclass of `UIView` for its rendering surface and is controlled as usual by the `UIViewController` but we need to add a few things and make a few chnages to start using SpriteKit:  

*   We need to open references for `MonoTouch.SpriteKit` and `MonoTouch.CoreGraphics`.
*   Add a couple of virtual overloads for `ViewDidAppear` and `ViewDidDisappear`.
*   Add an instance of `SKScene` and `SKView`.  
*   We will also add a function called `setupScene` which will initialise the scene, this will be called from `DidViewLoad`.

That means something like this will do the trick for an empty scene:  

{% codeblock lang:fsharp%}
namespace SpriteKitSingleView

open System
open System.Drawing
open MonoTouch.Foundation
open MonoTouch.UIKit
open MonoTouch.SpriteKit
open MonoTouch.CoreGraphics

[<Register ("SpriteKitViewController")>]
type SpriteKitViewController () as x=
    inherit UIViewController ()
    
    let mutable scene = Unchecked.defaultof<SKScene>
    let mutable spriteView = new SKView()
    
    let setupScene() =
        spriteView.Bounds <- RectangleF(0.f, 0.f, x.View.Bounds.Width * UIScreen.MainScreen.Scale, 
                                                  x.View.Bounds.Height * UIScreen.MainScreen.Scale)
        spriteView.ShowsDrawCount <- true
        spriteView.ShowsNodeCount <- true
        spriteView.ShowsFPS <- true
            
        x.View <- spriteView
        scene <- new SKScene (spriteView.Bounds.Size, 
                              BackgroundColor = UIColor.Blue,
                              ScaleMode = SKSceneScaleMode.AspectFit)

    override x.DidReceiveMemoryWarning () =
        base.DidReceiveMemoryWarning ()

    override x.ShouldAutorotateToInterfaceOrientation (orientation) =
        orientation <> UIInterfaceOrientation.PortraitUpsideDown
        
    override x.ViewDidLoad () =
        base.ViewDidLoad()
        setupScene()        

    override x.ViewDidAppear(animated) =
        base.ViewDidDisappear (animated)
        spriteView.PresentScene(scene)
        
    override x.ViewDidDisappear(animated) =
        base.ViewDidDisappear (animated)
        scene.RemoveAllChildren()
        scene.RemoveAllActions() 
{% endcodeblock%}

The main interesting bit here is `setupScene`.  The `spriteView` instance is created at the beginning of the `SpriteKitViewController's` [implicit constructor][3].  We need to defer creating the `scene` until later because in the constructor the `UIView` has not yet been initialised by the the framework and we would get a null reference exception.  

The first thing we do is get the dimensions of the current view, multiply it by the current scale, then apply it to the `spriteView` `Bounds` property.  The scale property is used for the various DPI modes in iOS devices.  Next we add a few debug outputs to the `spriteView` to show the current draw and node counts as well at the current frame rate.  We assign the `spriteView` to the `View` property of the `ViewController`.  And finally we create a new `SKScene`, assigning the view boundary, background colour and the scaling mode.  

As a side note we could create a storyboard and use an `SKView` as the custom class instead of the default `UIView`, doing this way means that when the `ViewDidLoad` overload is called the `View` property of the `SpriteKitViewControl` would already be initialized with a `SKView`.  This is a little more tricky in F# as we don't currently have the fancy UI designer integration in Xamarin Studio.  You would have to do this in Xcode and copy it to your project manually.  

###Adding the Sprite

Next we will add a spaceship sprite:  

*   We need to add a `.png` file to the project.
*   Create an instance of an `SKSprite`.
*   We also need to add the sprite to the scene.

Add a spaceship texture to the project and make sure that the build action is set to `BundleResource`

Next add the following to code to the `setupScene` function:  

{% codeblock lang:fsharp %}
use sprite = new SKSpriteNode ("Art/viper_mark_vii.png")
sprite.Position <- PointF (scene.Frame.GetMidX(), scene.Frame.GetMidY())
sprite.Name <- "Ship"
scene.AddChild(sprite)
{%endcodeblock%}

I added my sprite to a sub folder in the project called Art, once you start adding lots of graphics assets you probably want to make sure they are properly organised.  Next I set the spaceship's initial position to the center of the current scene using the `scene.Frame.GetMidX()` and `scene.Frame.GetMidY()` methods.  We give the sprite node a name using the `Name` property.  This is useful if we want to refer to the spaceship via its name in the node graph rather than using its object reference  Finally we add the sprite to the scene using `scene.AddChild(sprite)`.  

So this now gives us a single spaceship sitting in the middle of the screen:  

{% img https://lh4.googleusercontent.com/-P_PyM1pMkNE/Uj22UP6XaOI/AAAAAAAABs4/QhrH1JVRbSA/w592-h1236-no/scene+with+spaceship.png %}

###Creating Particles with xCode's Particle Designer

The great thing about SpriteKit is it also comes with a nice particle designer.  To use the particle designer you have to fire up Xcode and add a new file of type `SpriteKit Particle Designer`.  You get a choice of a eight different preset's particle types:  Bokeh, Fire, Fireflies, Magic, Rain, Smoke, Snow, and Spark.  

It certainly saves a lot of time developing the particle effects with the particle designer, there are loads of parameters to play around with.  If I quickly choose the rain preset and fiddle with the parameters a bit you get a star-field type effect like this:   

{% img https://lh4.googleusercontent.com/-8vyHp_i429s/Uj3B9SeUU9I/AAAAAAAABtk/GVOuegVGVlw/w1370-h1114-no/starfield.png %}

While we're here lets also create an exhaust plume for our spaceship, create another particle, this time using the spark preset and tweak it so it look a bit like this:

{% img https://lh3.googleusercontent.com/-3F9wo4u_uUI/Uj3B9XopZeI/AAAAAAAABtg/eIDF-kRuynQ/w1370-h1114-no/exhaust.png %}

###Adding The Particles 
We now have all we need to plug the particles into our demo.  Find the particles you created in Xcode and copy or move them into your project, don't forget to make sure that the build action to `BundleResource`.  

While I remember lets change the background colour to black, the blue looks a bit lurid and the exhaust trail wont look its best against a blue background.  Find where `scene` is initialised in `setupScene` and change it so it looks like this:  

{% codeblock lang:fsharp %}
scene <- new SKScene (spriteView.Bounds.Size, 
                      BackgroundColor = UIColor.Black,
                      ScaleMode = SKSceneScaleMode.AspectFit)
{% endcodeblock %}

If you look at the `SKEmitterNode` constructor you might be slightly befuddled by the fact that it only takes either an `NSCoder`, `NSObjectFlag` or a `nativeint`.   To help us out we create a nice little function to do the dirty work for us:

{% codeblock lang:fsharp %}
module spritekit =
    type SKEmitterNode with
        static member fromResource res =
            let emitterpath = NSBundle.MainBundle.PathForResource (res, "sks")
            NSKeyedUnarchiver.UnarchiveFile(emitterpath) :?> SKEmitterNode
{% endcodeblock %}

The `.sks` files produced by Xcode are archive files so we need to get them into a format that works in our project.  First we find the full path for the resource as its embedded in our app bundle - `NSBundle.MainBundle.PathForResource (res, "sks")`, next we use the  `UnarchiveFile` method from the `NSKeyedUnarchiver` type to get an `NSObject`.  We finally cast the `NSObject` as an `SKEmitterNode` before it is returned.   

The function shown above is added as a static extension method to the `SKEmitterNode`.  We could of also added this using a module or even just a simple function but at the moment we don't have a clear view of any other extensions that we might need so we'll just keep it tucked up in the `SKEmitterNode` type for now.  

We can now use this function to load and add our star-field to our scene.  Add this piece of code before the spaceship code we added previously:  

{% codeblock lang:fsharp %}
use stars = SKEmitterNode.fromResource "Stars"
stars.Position <- PointF(scene.Frame.GetMidX(), scene.Frame.GetMaxY())
scene.AddChild(stars)
{% endcodeblock %}

Its pretty simple now we have the helper function to load the star-field as a resource.  Notice we add the star-field as a child of the scene and position it at the middle X coordinate of the screen (`scene.Frame.GetMidX()`) and the maximum Y coordinate (`scene.Frame.GetMaxY()`).  This places the star-field centrally at the top of the screen.    

We can now go ahead and add our exhaust plume in the same way:  

{% codeblock lang:fsharp %}
use flame = SKEmitterNode.fromResource "Fire"
flame.Position <- PointF(0.f, -60.f)
sprite.AddChild(flame)
{% endcodeblock %}

The only difference here is we position the exhaust at location X = 0.0, Y = -60.0 and add the exhaust as a child of the spaceship.  This means that the exhaust is offset -60.0 in the Y axis from the spaceships location, this is because child nodes inherit their parents coordinate system.  This makes groups of sprites easy to animate and manipulate as you don't have to work out all the offsets.  

If we run our demo now it starts to look more interesting:

{% img https://lh6.googleusercontent.com/-cyiCu4dZOW4/Uj3QKqZINOI/AAAAAAAABuE/FAoU_bizdUk/w698-h1238-no/exhaust+and+starfield.png %}

That's all for now, I hope you have enjoyed this little look at particles in SpriteKit.  If you want to download the demo project you can find it in my [GitHub repo][4].

Until next time!  

* * *
####Essential listening:
*   Foo Fighters - Wasting Light  
*   Alice In Chains - Unplugged  
*   Pearl Jam - Ten  

    {% img http://upload.wikimedia.org/wikipedia/en/0/05/Foo_Fighters_Wasting_Light_Album_Cover.jpg 125 Foo Fighters - Wasting Light  %}
    {% img http://upload.wikimedia.org/wikipedia/en/4/43/AIC_Unplugged.jpg 125 Alice In Chains - Unplugged %}
    {% img http://upload.wikimedia.org/wikipedia/en/2/2d/PearlJam-Ten2.jpg 125 Pearl Jam - Ten %}

 [1]: https://developer.apple.com/library/ios/documentation/GraphicsAnimation/Conceptual/SpriteKit_PG/Introduction/Introduction.html
 [2]: http://box2d.org
 [3]: http://msdn.microsoft.com/en-us/library/dd233192.aspx
 [4]: https://github.com/7sharp9/SpriteKit-Fsharp-Samples
 [5]: http://xamarin.com
 [6]: http://blog.xamarin.com/ios-7-and-xamarin-ready-when-you-are/
 [8]: https://developer.apple.com/library/mac/documentation/graphicsimaging/Conceptual/CoreImaging/ci_intro/ci_intro.html
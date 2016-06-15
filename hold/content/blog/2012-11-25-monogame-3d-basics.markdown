---
author: 7sharp9
layout: post
title: "MonoGame 3D basics"
date: 2012-11-25
comments: true
categories: [Programming]
tags: [F#, tools, Mono, MonoDevelop, MonoGame, Mac]
---
{% img left http://download-codeplex.sec.s-msft.com/Download?ProjectName=monogame&DownloadId=331456&Build=19612 %}
This time we are going to get into a little bit of code and produce the simplest of all 3d solids, the 
tetrahedron.  I know its not the most exciting of things but we have to start somewhere.  The scope of 3D 
graphics in computers is so vast that its very easy to get lost in the vast piles of research papers.  

<!-- more -->

First lets do some basic setup, if you followed my last post then you will will have a project template to use, this 
makes this a little easier.  

**For those of you that are running on Windows and want to use Visual Studio please leave a comment if you would like 
a project template for F#**.  The beauty of MonoGame is that it is cross platform and there is only a small amount of code 
that differers between the different platforms, and that is localised to the main entry point rather than the `Game` type.  

First create a new `MonoGame Mac Application project`, you should end up with a Game1 type that looks like this:

{% codeblock lang:fsharp %}
type MonoGame3DBasics() as x =
    inherit Game()
    let graphics = new GraphicsDeviceManager(x)
    let mutable spriteBatch = Unchecked.defaultof<_>
    let mutable logoTexture = Unchecked.defaultof<_>
    do x.Content.RootDirectory <- "Content"
       graphics.IsFullScreen <- false
 
    /// Overridden from the base Game.Initialize. Once the GraphicsDevice is setup,
    /// we'll use the viewport to initialize some values.
    override x.Initialize() = base.Initialize()

    /// Load your graphics content.
    override x.LoadContent() =
        // Create a new SpriteBatch, which can be use to draw textures.
        spriteBatch <- new SpriteBatch (graphics.GraphicsDevice)
        
        // TODO: use this.Content to load your game content here eg.
        logoTexture <- x.Content.Load<_>("logo")

    /// Allows the game to run logic such as updating the world,
    /// checking for collisions, gathering input, and playing audio.
    override x.Update ( gameTime:GameTime) =
        // TODO: Add your update logic here                 
        base.Update (gameTime)

    /// This is called when the game should draw itself. 
    override x.Draw (gameTime:GameTime) =
        // Clear the backbuffer
        graphics.GraphicsDevice.Clear (Color.CornflowerBlue)

        spriteBatch.Begin()

        // draw the logo
        spriteBatch.Draw (logoTexture, Vector2 (130.f, 200.f), Color.White);
        spriteBatch.End()

        //TODO: Add your drawing code here
        base.Draw (gameTime)
{% endcodeblock %}

We are going to need a few extra field for this sample and we wont be using and 2d so remove the `spriteBatch` and the 
`logoTexture` as we wont be needing those.  The following fields need to be added in their place:

{% codeblock lang:fsharp %}
let mutable basicEffect = Unchecked.defaultof<_>
let mutable texture = Unchecked.defaultof<_>
let mutable vertexBuffer = Unchecked.defaultof<_>
let mutable view = Unchecked.defaultof<_>
let mutable world = Unchecked.defaultof<_>
let mutable projection = Unchecked.defaultof<_>
{% endcodeblock %}

What's this!  **mutable** fields! I know, but this simplifies things until I can put together a friendly functional 
scaffolding around [MonoGame][4].  We are creating a `basicEffect`, this is used to draw the 3D objects, its actually 
just a basic shader implementation with simple lighting.  We also have `texture` which will be used as our texture 
map.  We have a `vertexBuffer` which is used to store the vertices for out primitive.  `view`, `world`, and `projection` 
are our matrices which are used to look into our 3D scene.  For more information on the theory behind 3D projection have 
a look [here][3].  

Lets more to the `LoadContent` override:

{% codeblock lang:fsharp %}
override x.LoadContent() =
    //load texture
    texture <- x.Content.Load<Texture2D>("Tetrahedron")
    
    //world, view, projection
    world <- Matrix.Identity
    view <- Matrix.CreateLookAt(Vector3(0.f, 0.f, 10.f), Vector3.Zero, Vector3.Up)
    projection <- Matrix.CreatePerspectiveFieldOfView(MathHelper.PiOver4,
                                                      x.GraphicsDevice.Viewport.AspectRatio,
                                                      1.f,
                                                      1000.f)
        
    basicEffect <- new BasicEffect(x.GraphicsDevice,
                                   World = world,
                                   View = view,
                                   Projection = projection,
                                   Texture = texture,
                                   TextureEnabled = true)
                                     
    let tetrahedronData = generateTetrahedron 3.5f
    vertexBuffer <- new VertexBuffer(x.GraphicsDevice, 
                                     VertexPositionTexture.VertexDeclaration, 
                                     tetrahedronData.Length, 
                                     BufferUsage.WriteOnly)
    vertexBuffer.SetData(tetrahedronData)
    x.GraphicsDevice.SetVertexBuffer(vertexBuffer)
{% endcodeblock %}

The first thing that we do is load the texture map:  
`texture <- x.Content.Load<Texture2D>("Tetrahedron")`  
This simply loads in the texture named `Tetrahedron` using the content loader.  

Next we set up the default values for the world matrix, view and projection matrices.  The `world` is simply initialised using 
`Matrix.Identity` which is a matrix defined as: 
```
[1,0,0,0]
[0,1,0,0]
[0,0,1,0]
[0,0,0,1]
```

The `view` is initialised using the `CreateLookAt` method of the Matrix type.  This sets up a transformation that points from 
0,0,10 to the centre of the world using `Vector3.Zero`.  It also uses the `Vector3.Up` as the orientation direction *(Positive Y is up)*.   

The `projection` is also initialised using the helper method `CreatePerspectiveFieldOfView` which as you might guess, creates 
a perspective with a field of view.  In this instance our field of view uses the constant `PiOver4`.  

The basic effect is now initialised with the matrices we just initialised.  

For now I want you to ignore the line `let tetrahedronData = generateTetrahedron 3.5f`.  I need to explain how to generate a 
tetrahedron before that will make sense, just assume that is returns the vertices that we need for the tetrahedron.

The `vertexBuffer` is now created which will hold all the vertices for the tetrahedron.  We need to tell the `vertexBuufer` 
what format we want to use to hold the vertices, here, we are going to use vertices with Position, Colour, and Texture 
coordinates so we we use the predefined format of `VertexPositionTexture.VertexDeclaration`.  There are various different 
predefined formats and its also possible to create custom user defined formats, for more information have a look 
[here][5].  I realise I'm glossing over a lot of information, this is because the field of 3D graphics is huge even an API 
such as XNA/MonoGame which tries to simplify things, there is still a vast array of different concepts and I don't want to 
get too bogged down with all the specifics.  

Finally the `vertexBuffer` is assigned to the graphics device: `x.GraphicsDevice.SetVertexBuffer(vertexBuffer)`, this 
loads the vertex buffer into the graphics card ready to be draw later.

Next we move on to the `Update` override:

{% codeblock lang:fsharp %}
        override x.Update(gameTime) = 
            if Keyboard.GetState().IsKeyDown(Keys.Escape) then x.Exit()
            
            let time = float32 gameTime.ElapsedGameTime.TotalSeconds

            // Compute camera matrices.
            let rotationz = Matrix.CreateRotationY(time * 1.2f)
            basicEffect.View <- rotationz * Matrix.CreateLookAt(Vector3(0.f, 0.f, 10.f), Vector3.Zero, Vector3.Up)
                    
            base.Update (gameTime)
{% endcodeblock %}

The [`Update`][6] method is called every time the game decides that game logic needs to be processed. This includes the 
management of game state, the processing of user input, and also the updating of simulation data or AI.  

First of all we check the Escape key has been pressed so that the application can exit: `if Keyboard.GetState().IsKeyDown(Keys.Escape) then x.Exit()`.  

Next we capture the amount of elapsed time since the last update so that we can calculate distance moved over time etc.  

To make our view of the world less static we create a rotation around the z axis of the world so that we see the tetrahedron 
from different angles.   We multiply the rotation matrix by our initial `Matrix.CreateLookAt...` that we used earlier on, and 
assign it back to the `View` property of the `basicEffect`.   I want to stress that the aim of this is not super optimal code 
it's merely to show the easiest possible method of achieving a result.  In a future post we will be looking at some functional 
scaffolding to allow us to apply functional thinking to this domain.  Perhaps introducing a small [Domain Specific Languagee][10] to help.  

Finally we have the `Draw` override:

{% codeblock lang:fsharp %}
        /// This is called when the game should draw itself. 
        override x.Draw (gameTime) =
            // Clear the backbuffer
            x.GraphicsDevice.Clear (Color.CornflowerBlue)

            for pass in basicEffect.CurrentTechnique.Passes do
                pass.Apply()
                x.GraphicsDevice.DrawPrimitives(PrimitiveType.TriangleList, 0, 4)
                 
            base.Draw (gameTime)
{% endcodeblock %}

The [`Draw`][7] override is called every time the game needs to draw a frame, we put all out rendering code in here.

The first step is to clear the screen to a nice blue colour:

`x.GraphicsDevice.Clear (Color.CornflowerBlue)`

To draw our tetrahedron we need to loop through the different techniques in out shader (In this instance our basicEffect only has 1), apply the technique, then draw out triangles.  You might remember earlier to created a the `vertexBuffer` and assigned it to the graphics device.  All we have to do is tell MonoGame that we want to draw 4 triangles and they are in a `TriangleList`.  

That's it all done!  Well almost, now lets backtrack slightly and look at how we build the vertices for that tetrahedron.  

##Building a tetrahedron
What is a tetrahedron?  Well if you look on [wikipedia][1] 

{% blockquote %}
A tetrahedron is a polyhedron composed of four triangular faces, three of which meet at each vertex. It has six edges and four vertices. The tetrahedron is the only convex polyhedron that has four faces.
...
In the case of a tetrahedron the base is a triangle(any of the four faces can be considered the base), so a tetrahedron is also known as a "triangular pyramid".
...
For any tetrahedron there exists a sphere (the circumsphere) such that the tetrahedron's vertices lie on the sphere's surface.
{% endblockquote %}

The tetrahedron is also the simplest of the five [platonic solids][2].  There are lots of interesting properties of these
but I don't really want to go into that here we just want to draw and texture one for now.

So how do we construct a tetrahedron?

There are various methods that can be used to construct a tetrahedron ranging from formula such as:

Cartesian coordinate based
(±1, 0, -1/sqrt2)
(0, ±1, 1/sqrt2)

V0 =(0,0,1)
V1=(2sqrt2 /3, 0, −1/3)  
V2 =(− sqrt2 /3, sqrt6 /3, −1/3)  
V3=(− sqrt2 /3,− sqrt6 /3,−1/3)  

Yes I know I need to get latex maths expression working in my blog!  Ill have to work on that.

I don't know about you but I always feel uneasy unless I can clearly see exactly what's been done, I also don't 
want to turn this into a 3D geometry lesson because that's not what I intend this post to be about.

Here's what works for me anyway.

Calculate the radius of the circumsphere, this is the sphere in which all of the vertices of the tetrahedron sit, 
this is calculated by sqrt 3/8.  

The angle between each vertex and its centre point is acos -1/3 or ~ 109.471 degrees.  

*    The first vertex is (0, sqrt (3/8) * length, 0)
*    To get our second vertex we need to rotate the first vertex by acos -1/3 in the X axis
*    To get the third vertex we rotate second vertex by 120 degrees in the Y axis
*    For the last vertex we rotate the second vertex by -120 degrees in the Y axis

A picture can often be a worth a thousand words, I think this is one of those times, I will refer you to 
[Friedrich A. Lohmüllers site][8] for an excellent pictorial and description.  The code for this process is below.

{% codeblock lang:fsharp %}
let generateTetrahedron size = 
    let circumSphereRadius = sqrt (3.f/8.f) * size
    let centerVertexAngle = acos (-1.f/3.f)
    let v1 = Vector3(0.f, circumSphereRadius, 0.f)
    let v2 = v1 |> Vector3.rotateX centerVertexAngle
    let v3 = v2 |> Vector3.rotateY (radians 120.f)
    let v4 = v2 |> Vector3.rotateY (-radians 120.f)
    
    let uv1 = Vector2(0.5f, 1.f - sqrt 0.75f)
    let uv2 = Vector2(0.75f, 1.f - (sqrt 0.75f)/2.f)
    let uv3 = Vector2(0.25f, 1.f - (sqrt 0.75f)/2.f)
    let uv4 = Vector2(0.5f, 1.f)
    let uv5 = Vector2.UnitY
    let uv6 = Vector2.One

    [| VertexPositionTexture(v1, uv1)
       VertexPositionTexture(v3, uv2)
       VertexPositionTexture(v2, uv3)  
       
       VertexPositionTexture(v1, uv2)
       VertexPositionTexture(v4, uv6)
       VertexPositionTexture(v3, uv4)

       VertexPositionTexture(v1, uv3)
       VertexPositionTexture(v2, uv4)
       VertexPositionTexture(v4, uv5)

       VertexPositionTexture(v2, uv3)
       VertexPositionTexture(v3, uv2)
       VertexPositionTexture(v4, uv4) |] 
{% endcodeblock %}

The last piece of the puzzle is the texture coordinates.  There is some amazing software available to help model both texture 
and 3d geometry, projecting the vertices onto a 2d plane can be an art-form in itself.  Luckily the tetrahedron is one 
of the simplest models, if you imagine the tetrahedron unfolded it would look like this from the top:

{% img http://upload.wikimedia.org/wikipedia/commons/thumb/8/88/Tetrahedron_flat.svg/240px-Tetrahedron_flat.svg.png 250 %}

To map a texture to the tetrahedron we have to include a texture coordinate with every vertex.  These coordinates are `uv1-uv6` 
in the code above.  We use some ratios to select the correct coordinates within the texture.  The texture coordinates are always 
between 0 and 1.  Here's the location of the above points so you can see the locations clearly.

{% img https://lh4.googleusercontent.com/-NpV1nz3K-kI/ULKY-KY6emI/AAAAAAAABio/565nXCNC9xY/s425/texture+coords.png 250 %}

To make sure that the texture is in the right place we are going to use a type of fractal called the 
[Sierpinski triangle][9].  The Sierpinski triangle had exactly the same net, or unfolded shape as the texture we need to use.  Each 
of the first iterations of the fractal is coloured separately as this will make it easy to see if the mapping is  correct.  Here 
is what the texture looks like:  

{% img https://lh4.googleusercontent.com/-hLyy6qGXjWc/ULKn_4p-CUI/AAAAAAAABjA/azvh8cUNrAY/s310/Tetrahedron.png 250 %}

This is how everything will look, I know its not incredibly impressive but its [MonoGame][4] in 3d, F#, and all running on a 
Mac, what more could you want! ;)  

{% img https://lh5.googleusercontent.com/-P_7wHvJbdY0/ULO2xNStVwI/AAAAAAAABjQ/SGA5DT4ChgU/s800/Screen+Shot+2012-11-26+at+18.33.57.png %}

Its feels like we covered a lot of ground here but all we have is a spinning tetrahedron, its tricky to know what level of 
detail to go down to.  I don't want to teach anyone how to suck eggs, and I want to alienate people who are new 
to this area and want to learn, I hope I got the balance about right.  

If you want to just get the code and have a look then here's my [GitHub repo][11]

As usual I appreciate any comments and feedback.  

Until next time!

[1]:http://en.wikipedia.org/wiki/Tetrahedron
[2]:http://en.wikipedia.org/wiki/Platonic_solid
[3]:http://en.wikipedia.org/wiki/3D_projection
[4]:http://monogame.codeplex.com
[5]:http://devblog.phillipspiess.com/2010/03/21/xnas-vertex-structs-and-custom-vertex-formats/
[6]:http://msdn.microsoft.com/en-us/library/microsoft.xna.framework.game.update.aspx
[7]:http://msdn.microsoft.com/en-us/library/microsoft.xna.framework.game.draw.aspx
[8]:http://www.f-lohmueller.de/pov_tut/x_sam/sam_440e.htm
[9]:http://en.wikipedia.org/wiki/Sierpinski_triangle
[10]:http://en.wikipedia.org/wiki/Domain-specific_language
[11]:https://github.com/7sharp9/PlatonicSolids
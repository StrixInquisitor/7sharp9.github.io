+++
author = "7sharp9"
categories = ["programming"]
tags = ["Elm", "Functional Programming"]
date = "2016-06-30T13:53:45+01:00"
description = "Getting your head around Elm 17 updates"
title = "Elmtastic Updates"
type = "post"
images = ["http://7sharpnine.com/img/elmlogo.png"]

+++
With the release of Elm 0.17 there were some fundamental changes to the Elm language.  This post 
is my attempt to help those that may be struggling with these changes<!--more-->

I've played with lots of new languages over the last year or so namely Elixir, Rust, and Elm.  Elm 
and Elixir have been my favorites and I hope to cover those much more in this and future blog posts.  

# So whats new in Elm 0.17 why do I need to upgrade anything?

I'll summarise here:

* Signals have been removed (Hence some upgrade is required if you had any code using signals.)
* Faster HTML renderer
* Libraries for geolocation, page visibility, and web sockets
* Generated JS is smaller and works with Google's Closure Compiler
* Generated JS works with RequireJS and CommonJS
* Features in place for services like GraphQL and Elixir Phoenix
* Improved documentation at guide.elm-lang.org
* Helpful messages when decoding JSON fails

The big things are that are going to throw a spanner in the works are Signals have been removed 
and the following packages have moved around:  

```bash
evancz/elm-html    -> elm-lang/html
evancz/elm-svg     -> elm-lang/svg
evancz/virtual-dom -> elm-lang/virtual-dom
evancz/start-app   -> elm-lang/html
evancz/elm-effects -> elm-lang/core
```

Be sure to read the [official post][1] on the subject for the full information.  

## How Do I upgrade?  Are there any resources to help?

There are a few guides already that do help, the [official upgrade plan][2] is 
really useful as is [migrating from elm 0.16 to 0.17][3] and I would advise that 
you read the official plan before this or any other guides.  Hopefully my whistle stop 
tour of upgrading an existing package may be of help.  

## Example Upgrade

Lets take the [elm-sprite][5] package as an example, its fairly simple with only a few 
dependencies.  

### elm.package.json
Lets look at the elm.package.json file:

```json
{
    "version": "1.0.0",
    "summary": "Simple sprite rendering for elm-html",
    "repository": "https://github.com/Fresheyeball/elm-sprite.git",
    "license": "MIT",
    "source-directories": [
        "src"
    ],
    "exposed-modules": [
        "Sprite"
    ],
    "dependencies": {
        "elm-lang/core": "3.0.0 <= v < 4.0.0"
    },
    "elm-version": "0.16.0 <= v < 0.17.0"
}
```

We need to update the dependencies for `elm-lang/core` to `4.0.1 <= v < 5.0.0` 
and `elm-version` to `0.17.0 <= v < 0.18.0`.  Pretty easy in terms of dependencies.  

### Sprite.elm

This ones pretty easy, the only thing to change is the module definition which is 
using an obsolete syntax:    

```elm
module Sprite (..) where
```
Now becomes
```elm
module Sprite exposing (..)
 ```

 Actually the Elm compiler does a fantastic job here by actually telling us what the problem is:

```ini
-- SYNTAX PROBLEM --------------------------------------------------- Sprite.elm

I ran into something unexpected when parsing your code!

1| module Sprite (..) where
                 ^
I am looking for one of the following things:

    something like `exposing (..)` which replaced `where` in 0.17
    whitespace
```
		
So that was pretty painless, lets have a look at the example file: `One.elm` 
### One.elm

This ones a bit more tricky as there are signals involved and packages that have moved about.  
Lets have a look at the changes needed.

First of all lets address the obsolete `where` syntax:
```elm
module One exposing (..)
```

We need to remove the `Signal`, `Html.Events` and `Effects` imports packages, let's remove these:  
```elm
import Signal exposing (message, Address)
import Html.Events exposing (on, targetValue)
import Effects exposing (Effects, none)
 ```

Now we need to address the changes in the Time package as `fps` is no longer available, we can use milliseconds instead.  
```elm
import Time exposing (Time, millisecond)
```

Now we need to adjust the `StartpApp` import and use `Html.App` instead.  
```elm
import Html.App as Html
```

So all in all the imports section will look like this:

```elm
module One exposing (..)

import Html exposing (..)
import Html.App as Html
import Time exposing (Time, millisecond)
import Html.Attributes as A
import Sprite exposing (..)
import Array
```
### Action
The first thing we will tackle is the Action which flows though this application.  It now looks like this:
```elm
type Action
    = Tick Time
```

As `Action` has now been replaced with `Msg` so we need to change this to the following:
```elm
type Msg
  = Tick Time
```

### init
Next lets look at the init function its signature is slightly different now.  `Html.Program` now [starts the application][4]

```elm
init : (model, Cmd msg)
```

So rather than:

```elm
init : Sprite {}
init =
   { sheet = "https://10firstgames.files.wordpress.com/2012/02/actionstashhd.png"
     , rows = 16
     , columns = 16
     , size = ( 2048, 2048 )
     , frame = 0
     , dope = idle
     }
```

It will now become:  
```elm
init : (Sprite {}, Cmd Msg)
init = (
  { sheet = "https://10firstgames.files.wordpress.com/2012/02/actionstashhd.png"
    , rows = 16
    , columns = 16
    , size = ( 2048, 2048 )
    , frame = 0
    , dope = idle
    } , Cmd.none)
 ```

### view
This is what view looks like:
```elm
view : Address Action -> Sprite {} -> Html
view address s =
    let
        onInput address contentToValue =
            on
                "input"
                targetValue
                (message address << contentToValue)
    in
        div
            []
            [ node
                "sprite"
                [ A.style (sprite s) ]
                []
            ]
```

At first glance this looks a bit more complex but when you look at the code a little 
more you come to realise that the `onInput` function is not even used anymore this is 
just dead code.  So no all that remains is to change `view` to match the new Elm 0.17 
architecture, so instead of `Address Action -> Sprite {} -> Html` it will now be 
`Sprite {} -> Html Msg` as `Address` and `Action` are now no longer needed.  
```elm
view : Sprite {} -> Html Msg
view s =
  div
      []
      [
        node
          "sprite"
          [ A.style (sprite s)]
          []
      ]
```

### update
Ok, now for `update` lets have a look at that:
```elm
update : Action -> Sprite {} -> ( Sprite {}, Effects Action )
update action s =
    let
        s' =
            case action of
                Tick _ ->
                    advance s
    in
        ( s', none )
```

`update` now has a signature of `msg -> model -> (model, Cmd msg)` so all we really have 
to do is replace `Action` with `Msg`, and `Effects Action` with `Cmd Msg`.  Finally I 
change the return to: `Cmd.none` which was previously `Effects.none`.  

```elm
update : Msg -> Sprite {} -> (Sprite {}, Cmd Msg)
update action s =
    let
        s' =
            case action of
                Tick _ ->
                    advance s
    in
        ( s', Cmd.none )
```

### subs
This part is new, with the old 0.16 based version there was a signal which was mapping 
time to a sprite update:`[ Signal.map Tick (fps 30) ]`.  Now that we will be using 
subscriptions this is a simple function like this:

```elm
subs : Sprite {} -> Sub Msg
subs model =
  Time.every (millisecond * 33) Tick
```

So every 33 milliseconds (30 frames per second) we are sending a Tick command to the update function.

### application start
The last part is the application start, heres what it looks like in 0.16:
```elm
app : StartApp.App (Sprite {})
app =
    StartApp.start
        { view = view
        , update = update
        , init = ( init, none )
        , inputs = [ Signal.map Tick (fps 30) ]
        }


main : Signal Html
main =
    app.html
```
You can see the `signal` I talked about above.  This whole section has now become a lot simpler in Elm 0.17.

```elm
main : Program Never
main =
  Html.program
    { view = view
    , update = update
    , init = init
    , subscriptions = subs
    }
```

`StartApp` has now become `Html.App` which we aliased to `Html` at the beginning `import Html.App as Html` 
and we use the `program` function to feed in all the functions we just declared.  

Ok, we're all done, hopefully someone found this useful!  

One tip I can give is to update the function signatures for each function first and it should make 
things a little clearer on what you need to do.  

Until next time!
- - -
[1]: http://elm-lang.org/blog/farewell-to-frp
[2]: https://github.com/elm-lang/elm-platform/blob/master/upgrade-docs/0.17.md
[3]: http://www.lambdacat.com/migrating-from-elm-0-16-to-0-17-from-startapp/
[4]: http://package.elm-lang.org/packages/elm-lang/html/1.1.0/Html-App#program
[5]: https://github.com/Fresheyeball/elm-sprite

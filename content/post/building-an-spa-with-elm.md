+++
author = "7sharp9"
date = "2016-10-03T00:32:48+01:00"
description = "Building a single page application with Elm"
tags = ["Elm", "SPA", "SEO", "FSharp", "Fable"]
title = "Building a single page application with Elm"
+++
I've been meaning to write this for a while but got sidetracked with other things, so this is an experience report of 
using Elm to build a single page application.  <!--more-->

First of all you can see the final site [here][1]

# Basic Structure
The basic structure is a navigation driven single page application, which also uses [The Elm Architecture][2]

I split the site into separate file mainly based on [how i structure elm apps][3] by [Kris Jenkins][4].

```bash
├─ App.elm
├─ Data.elm
├─ Shared.elm
├─ State.elm
├─ Types.elm
├─ View.elm
│
├─ About
│  └─ View.elm
├─ Category
│  └─ View.elm
├─ Contact
│  └─ View.elm
├─ Detail
│  └─ View.elm
├─ Gallery
│  └─ View.elm
└─ Home
   └─ View.elm
```

I think its slightly overkill for a site of this size and complexity but its nice to understand where things will lead 
on a bigger site where you have additional files like `State.elm`, `Types.elm` and `Rest.elm` etc.  

One of the things that I did find a annoying at times was if I had `About/View.elm` and `Category/View.elm` both open in 
the tabs of my editor.  Only `View.elm` is shown in the tab so I had to hovering on the tab to see the full path or read 
the module name at the top of the file.  Renaming the file to `AboutView.elm` would mean the import statement would have 
to change to `import About.AboutView` as Elm enforces the directory name prefix.   

# Application structure

As per the [how i structure elm apps][3] article my application was structured like this:
```elm
module Main exposing (..)
import View exposing (rootView)
import Navigation
import State


main : Program Never
main =
    Navigation.program (Navigation.makeParser State.pathParser)
        { init = State.init
        , view = View.rootView
        , update = State.update
        , urlUpdate = State.urlUpdate
        , subscriptions = State.subscriptions
        }
```

All the main elements of the application are imported and a `Navigation.program` is started with the appropriate `init`, 
`view`, `update`, `urlUpdate` and `subscriptions`.  

# Model Messages and types

## Model
The model is super simple, all that has to be recorded is the current page:
```elm
type alias Model =
    { page : Page }
```

## Messages
The messages the application will deal with are also pretty simple:
```elm
type Msg
    = NavigateTo Page
    | NavigateBack
    | GmailCaptcha
```
*  `NavigateTo` simply navigates to the page in question.
*  `NavigateBack` just navigates back one page.
*  `GmailCaptcha` displays a captcha request which then shows an email address on success.

Page is just a union type consisting of the pages in the application:
```elm
type Page
    = Home
    | About
    | Contact
    | Gallery
    | CategoryDetail CategoryType
    | ItemDetail String
```

And `CategoryType` another union:
```elm
type CategoryType
    = Perfumes
    | Seasides
    | IllustratedQuotes
    | Bottles
    | Cocktails
    | Architecture
    | VintageCameras
    | Cakes
    | Unknown
```

Category and Item are simple records:
```elm
type alias Category =
    {categoryType : CategoryType, img : String, description : String}

type alias Item =
    {id : String, title : String, img : String, description : String, category : CategoryType}

```

# update

All Elm application have an `update` function and ours is defined like this:
```elm
update : Msg -> Model -> ( Model, Cmd b )
update msg model =
    case msg of
        NavigateTo page ->
            ( model, (Navigation.newUrl <| pageToString page) )

        NavigateBack ->
            model => (Navigation.back 1)

        GmailCaptcha ->
            ( model, captcha() )
```
We pattern match on the `msg` and either use the commands described in [Messages]({{< ref "#messages" >}})

# View
The View is quite simple its a virtual dom defined by the following nodes on the `rootView:

```elm
rootView model =
    div [class "container"]
        [
            nav [ class "navbar navbar-light", attribute "role" "navigation" ]
                [ a [class "pull-xs-left"
                    , href <| toHash Home
                    , onClick' <| NavigateTo Home
                    ]
                    [img [id "logo", class "img-fluid", src "/img/logogreen.png", srcset ["/img/logogreen.png","/img/logogreen@2x.png" ] ] []]
                  ,  button [ attribute "aria-controls" "exCollapsingNavbar2"
                            , attribute "aria-expanded" "false"
                            , attribute "aria-label" "Toggle navigation"
                            , class "navbar-toggler hidden-sm-up flex-center"
                            , attribute "data-target" "#exCollapsingNavbar2"
                            , attribute "data-toggle" "collapse"
                            , type' "button"
                            ]
                                [ text "☰" ]
                        
                , div [ class "collapse navbar-toggleable-xs", id "exCollapsingNavbar2" ]
                    [   ul [ class "nav navbar-nav pull-sm-right text-xs-center" ]
                        [ renderMenuItem model Home "Home"
                        , renderMenuItem model About "About"
                        , renderMenuItem model Gallery "Gallery"
                        , renderMenuItem model Contact "Contact"
                        ]
                    ]
                ]
        , div [class "content container-fluid"] [viewPage model]
        , footer
        ]
```

`renderMenuItem` is just changing a navigations item's style based on the current page so I have omitted that.

`viewPage` is where the view is updated depending on which page is current:
```elm
viewPage model =
    case model.page of
        Home -> getHomePage ()
        About -> getAboutPage ()
        Gallery -> getGalleryAsCards ()
        Contact -> getContactPage () 
        CategoryDetail category -> getCategoryPageCards category
        ItemDetail item -> getItemPage item
```
You can see there is a separate view for each page which in return a list of nodes for that particular view, I wont list 
them all as the are relatively similar.

Heres an example of the `Category.View`:
```elm
getCategoryPageCards category =
    let
        items =
            List.filter (\c -> c.category == category) Data.items
        colClass =
            case List.length items of
                1 -> "col-xs-12"
                2 -> "col-xs-12 col-sm-6"
                _ -> "col-xs-12 col-sm-6 col-md-4"


        itemMapper item =
            div [ class colClass ]
                [ div [ class "card"]
                    [ a [ noContextMenu
                        , href (toHash <| ItemDetail item.id)
                        , onClick' (NavigateTo <| ItemDetail item.id)
                        ]
                        [ img [ noContextMenu, class "card-img-top img-fluid", src item.img ] [] ]
                    , div [ class "card-block" ]
                        [ p [ class "card-text" ] [ text item.title ] ]
                    ]
                ]
    in
        div [ class "container-fluid" ]
            [ div [ class "row" ]
                (items |> List.map itemMapper)
            , div [] [ backButton ]
            ]
```
The nodes returned from `getCategoryPageCards` are returned as part of `viewPage`.  Data.items are filtered by the current 
category and mapped into `divs` with `onClick` navigation to the `ItemDetail` page

# Navigation
Navigation is handled with Elm [Navigation][5] which provides the `Navigation.program` you saw in [Application structure]({{< ref "#application-structure" >}}).  
The program function has been extended with an additional two arguments.

The first additional argument is a `Parser`, there is a utility function called `makeParser that allows us to turn a 
browser `Location` into whatever data we want to.

```elm
makeParser : (Location -> a) -> Parser a
```

In the `Navigation.program` you can see this used along with the functions below to parse a `Location` into a `Page`: `Navigation.program (Navigation.makeParser State.pathParser)`

```elm
pathParser : Navigation.Location -> Result String Page
pathParser location =
    parse identity pageParser (String.dropLeft 1 location.pathname)


pageParser : UrlParser.Parser (Page -> a) a
pageParser =
    oneOf
        [ format Home (oneOf [ s "home", s "" ])
        , format About (s "about")
        , format Shop (s "shop")
        , format Gallery (s "gallery")
        , format Contact (s "contact")
        , format (stringToCategoryType >> CategoryDetail) (s "category" </> UrlParser.string)
        , format ItemDetail (s "item" </> UrlParser.string)
        ]
```

This leads nicely into parsing, actually first lets look at `urlUpdate` because its pretty simple:

```elm
urlUpdate : Result a Page -> Model -> ( Model, Cmd c )
urlUpdate result model =
    case result of
        Err _ ->
            ( model, Navigation.modifyUrl (pageToString model.page) )

        Ok page ->
            { model | page = page } => updateAnalytics (pageToString page)
```
We pattern match on the `result` and if its `Ok` we update the models page to the one passed in.  If the result is an 
error (`Err`) then we modify the url with `Navigation.modifyUrl` just pointing it back to the previous page.  `pageToString` 
simply turns the `Page` type back into a string.

I'm going to strategically ignore `updateAnalytics` 
for now as this will be covered later.  

# Parsing
Parsing is handled with a small parser combinator library [url-parser][6] in the function above you can see how combinators 
are used to parse `Navigation` into a `Page`

*  `oneOf` is a combinator that will try to match one of the parsers in a list.
*  `s` is a string combinator matching a particular string like "home", "shop" etc.
* `</>` is a combinator that matches a `/` character in the location like `item/myitem`.
* `UrlParser.string` matches any string.
*  `format` Is a combinator that allows you to customise or map another `Parser`, here it is used to Parsed output into the union types that represent them.  

# Google Analytics

Ok back to `updateAnalytics` remember from `urlUpdate`?
```elm
{ model | page = page } => updateAnalytics (pageToString page)
```

`updateAnalytics` is a function defined as follows:

```elm
port updateAnalytics: String -> Cmd msg
```
Yes its just a type signature as this is a [port][7] where we will be sending information out of our app into the fabulous 
JavaScript world, I'm glad you can't hear the sarcasm in my voice :-)

Because this is a single page application we need to provide a way to update Google analytics whenever the page navigation 
changes otherwise everything will be displayed as a hit to the root.  Luckily all we need is a call to the normal 
Google Analytics script via the port we just set up.

The JavaScript looks like this:
```JavaScript
var Elm = require('./App');
var app = Elm.Main.embed(document.getElementById('main'));

app.ports.updateAnalytics.subscribe(function (page) {
    ga('set', 'page', page);
    ga('send', 'pageview');
});
```

On the fist two lines you can see where Elm is embedded into the the html `main` element, the important part is where 
`app.ports.updateAnalytics.subscribe` is used to subscribe to the port Elm is publishing whenever `updateAnalytics` is 
being called with the new `Page`.  The result of this is we can now use Google Analytics to see exactly what page a user 
is visiting.  

Actually the `GmailCaptcha` message is the same except it uses a `Port` defined like this:
```elm
port captcha : () -> Cmd msg
```
And then the JavaScript subscription is as follows:
```JavaScript
app.ports.captcha.subscribe(function () {
    window.open('http://www.google.com/recaptcha/mailhide/...', '', 'toolbar=0,scrollbars=0,location=0,statusbar=0,menubar=0,resizable=0,width=500,height=300');
    return false;
});
```
All that's happening here is that when the `GmailCaptcha` message is sent a window is opened via the `Port` and JavaScript to allow 
the email address to be retrieved.

# Summary
This post never intended to explain in detail about creating the web application more just to expose some of the more 
interesting parts and to show how much fun it is working in Elm is.  

I used VSCode with the [elm addin][8] to build the entire thing and it worked out wonderfully.  

## Comparison with Fable
This is the bit I was dreading a little.  [Fable][9] is a F# -> JavaScript transpiler.  I've done quite a lot of hacking 
with Fable and also made some contributions lately.  I would describe Elm as batteries included, and very slick, and 
lovely to work in.  I really love using it!  I really like the great compiler message, I also like the public type 
annotations that the Elm compiler nudges you to add.  Its very useful when reading code on-line to know the types on the 
publicly exposed parts so that you don't have to use a visual editor to find out.  I could probably write an entire post 
comparing the two languages so I'll leave it there for now.  

# So whats Left?
All that's left is the deployment and hosting side which I'll delve into another time.  

If you have any requests for me to go into parts in more detail please leave a comment or ping me on Twitter and I'll add that in too.

Until next time!
- - -

[1]: http://lynseythomas.com
[2]: https://guide.elm-lang.org/architecture/
[3]: http://blog.jenkster.com/2016/04/how-i-structure-elm-apps.html
[4]: https://twitter.com/krisajenkins
[5]: http://package.elm-lang.org/packages/elm-lang/navigation/latest
[6]: http://package.elm-lang.org/packages/evancz/url-parser/latest/
[7]: https://guide.elm-lang.org/interop/javascript.html#ports
[8]: https://marketplace.visualstudio.com/items?itemName=sbrink.elm
[9]: https://fable-compiler.github.io/


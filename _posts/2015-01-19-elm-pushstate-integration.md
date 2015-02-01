---
layout: post
title: Integrating Elm with the browser address bar and html5 pushState
---

[Elm](http://elm-lang.org) is a functional language focused on making user interfaces which compile to javascript and makes highly dynamic webapps very easy to write correctly.  The libraries available handle some aspects of web programming (eg: Canvas graphics or HTML generation) very well with some nice APIs.  However, at the time of writing there are some gaps in the browser API for which you must drop down into javascript to access. This article aims to show how to nicely integrate Elm with `window.location` and the HTML5 browser history API.  I'm assuming you know what the browser history API is for and have had at least a cursory overview of Elm so that you understand it's syntax, Signals and Ports.

The API
=======

We'd like both a way to write to the browser location history, and also to read the initial location from the browser as the app starts up.  We're assuming for now that there's no other javascript running on the page that might set the location.

The standard way of manipulating a time-varying value in Elm is to use a `Signal`, and the standard library provides many signals to process user input, such as the `Window.dimensions`signal which gives you the width and height of the browser window.  However, these are read-only - the standard way to send a signal out of Elm is to use a `Port`.  That leads us to think that we need a constant incoming value giving the initial browser location, and a time-varying outgoing value.  For simplicity, we'll read the location in and write it out as a `String`.

````haskell
port initialLocation : String -- incoming
port location : Signal String -- outgoing
````

The App
=======

We can then link those up into our app to give the desired behaviour.  For example, to bind the address location to an `<input>` field:

````haskell
-- Set up a channel to submit textfield content, initialized with the initial location
textContent : Channel Content
textContent = channel {noContent | string <- initialLocation}

-- Style the field and render
main : Signal Element
main = field defaultStyle (send textContent) "Type here" <~ subscribe textContent
````

The Plumbing
============

All that now remains is to link our Elm app up to the actual history API using javascript. Use `elm-make` or `elm-reactor` to compile the .elm file into elm.js and add the following `index.html` page.  Here we are reading and writing from the query portion of the URL only, so that refreshing the page in `elm-reactor` doesn't result in a 404, but it should be clear how to extend this to cover the full url.

````html
<!DOCTYPE html>
<html>
  <head></head>
  <body>
  <div id="elm"><!-- We will insert the elm app here --></div>
    <script src="elm.js"></script>
    <script type="text/javascript">
      var div = document.getElementById('elm');

      // Embed the elm app into a div and pass the initial location
      var elmApp = Elm.embed(Elm.BrowserIntegrationExample, div,
        { initialLocation: window.location.search.substring(1) });

      // Listen to output on the location port and pass to browser
      elmApp.ports.location.subscribe(function (newValue) {
          history.replaceState({}, "", "?" + newValue);
      })
    </script>
    </div>
  </body>
</html>
````

[It works!](http://jazmit.github.io/elm-browser-integration-example)  The content of the text field is now bound to the location search.  Editing the text field will update the address bar, and changing the address and loading the page will update the text field.

This [roof calculator](http://jazmit.github.io/roof-calculator/?w=3.95&l=3.55&h=2) is a more realistic example using browser integration, source code available [here](https://github.com/jazmit/roof-calculator).

Here is the [complete example source](http://github.com/jazmit/elm-browser-integration-example) for this article.

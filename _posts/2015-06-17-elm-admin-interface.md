---
layout: post
title: Building a simple admin interface using elm
---

In the constant search of better technologies to develop web apps, I've been evaluating Elm for a while now and it shows a lot of promise.  Some (mainly Java / C#-heads) critize it for being too complex.  Some voices (mainly the hard-core Haskell crowd) critize it for lacking power when compared to other functional effects.  For me, it has a nice balance of simplicity and power which hits the sweet spot for complex asynchronous web interfaces.

However, a lot of web development is much simpler than this so I wanted to see if I could use Elm to build something simple quickly - does it make it easy to do the simple things?

The task is a simple admin interface for a web app I'm developing in Angular.  The interface needs to collect various fields for an 'AND' search for users (first names, last name, email, etc), then send an AJAX request to the server and display the JSON result (a list of users).  Simple enough task, and one that could be done quickly using any web framework or even without one, but it is the bread and butter of web development, and if Elm wants to hit the main stream it should be able to dispatch these kinds of task with aplomb.

Splitting it down into subtasks gives:

* Create and display input fields
* Read values from input fields on enter / submit click
* Send search as AJAX request
* Interpret JSON from AJAX response
* Display list of users


Create and display input fields
===============================

This is very straightforward using the declarative elm-html API.

````haskell
import Html exposing (..)

textSearchField : String -> Html
textSearchField fieldLabel =
    div [] [
        text fieldLabel,
        input [] []
    ]

inputs : Html
inputs =
    div [] [
        textSearchField "Email:",
        textSearchField "Firstnames:",
        textSearchField "Lastnames:"
    ]

main : Html
main = inputs
````

Read values from input fields
=============================

Now, following the Elm architecture, we create a type for updates to the application and a `Mailbox` to post updates to

````haskell
import Signal exposing (..)

type SearchUpdate
    = Email String -- Search emails
    | First String -- Search first names
    | Last String  -- Search surnames
    | Noop

searchUpdates : Mailbox SearchUpdate
searchUpdates = mailbox Noop
````

Now, how do we get the input fields to send events to the mailbox?  Examining the example TODO application's source code shows there is an `on` attribute which does the job, if we pass it a function which generates the updates.  In our case, the update-generating function are the contructors `Email`, `First`, etc.

````haskell
import Html.Attributes exposing (..)
import Html.Events exposing (..)

textSearchField : String -> (String -> SearchUpdate) -> Html
textSearchField fieldLabel generateUpdate =
    let makeMessage = message searchUpdates.address << generateUpdate
    in  div [] [
            text fieldLabel,
            input [ on "input" targetValue makeMessage ] []
        ]


inputs : Html
inputs =
    div [] [
        textSearchField "Email:" Email,
        textSearchField "First name:" First,
        textSearchField "Surname:" Last
    ]
````

Now, we have all those updates in a signal, but we need all fields simultaneously in order to actually perform a search.  We'll create a record type to represent all the fields and use `foldp` to combine the updates from individual fields.

````haskell
type alias Search = {
    first : String,
    last  : String,
    email : String
}


searches : Signal Search
searches =
    let step upd oldSearch =
            case upd of
                Email email -> { oldSearch | email <- email }
                First first -> { oldSearch | first <- first } 
                Last  last  -> { oldSearch | last  <- last  } 
                Noop        -> oldSearch
    in  foldp step { first="", last="", email="" } searchUpdates.signal
````

For good measure, we can add some debug output to check that everything works as it should.

````
debugOutput : Search -> Html
debugOutput search = text <| search.first ++ " " ++ search.last ++ " " ++ search.email

render : Search -> Html
render search =
    div [] [
        inputs,
        debugOutput
    ]

main : Signal Html
main = render <~ searches
````

This now works, and the debug output updates as we type. However, this seems a little over-complicated - we had to define Noop for the initial update, and also the default blank search record.  We create a special type for updates which is only then mapped to record fields in foldp.  A moments thought shows we can eliminate the `SearchUpdate` type, and simply pass functions which update the record as messages.  This is a departure from the recommended Elm architecture, but it simplifies things greatly:  

````haskell
type alias SearchUpdate = Search -> Search


searchUpdates : Mailbox SearchUpdate
searchUpdates = mailbox identity


searches : Signal Search
searches = foldp (<|) { first = "", last = "", email = "" } searchUpdates.signal

inputs : Html
inputs =
    div [] [
        textSearchField "Email:"       (\email search -> { search | email <- email}),
        textSearchField "First names:" (\first search -> { search | first <- first}),
        textSearchField "Surnames:"    (\last  search -> { search | last  <- last}),
        button [ id "submit-search" ] [
            text "Submit"
        ]
    ]
````

All the other code remains unchanged.  We can now use the identity function instead of the special Noop message, and adding a new field only requires changing inputs and adding a field to one default value.

The last thing we need before moving on to AJAX is to limit the updates to when the user hits enter.  We collect the enter events from the inputs and pass them to a new mailbox.  The mailbox has no type as we are only interested in when submits occur, not what kind they are.

````haskell
submits : Mailbox ()
submits = mailbox ()


onEnter : Message -> Attribute
onEnter msg =
    let is13 code =
            if code == 13
               then Ok ()
               else Err "wrong key code"
    in  on "keydown"
            (customDecoder keyCode is13)
            (\_ -> msg)


textSearchField : String -> (String -> SearchUpdate) -> Html
textSearchField fieldLabel generateUpdate =
    let makeUpdate = message searchUpdates.address << generateUpdate
    in  div [] [
            text fieldLabel,
            input [
                on "input"   targetValue makeUpdate,
                onEnter (message submits.address ())
            ] []
        ]
````

Here I 'borrowed' the onEnter utility function from the todo example.  All that remains now is to limit the searches signal to change only when a submit happens.  This is the purpose of `Signal.sampleOn`.

````haskell
searches : Signal Search
searches = sampleOn submits.signal <|
    foldp (<|) { first = "", last = "", email = ""} searchUpdates.signal
````

Sending the search as an AJAX request and decoding the result
=============================================================

Before issuing the http request, we need to be able to decode the response from JSON to an Elm type.  Elm has a mini-DSL for doing this in the form of the Json.Decoder library.  We just need to define an Elm type for our user and a decoder to get a list of users from JSON.

````haskell
import Json.Decode as Json exposing (..)

type alias User = {
    id      : Int,
    first   : String,
    last    : String,
    emails  : List String
}

decodeResponse : Json.Decoder (List User)
decodeResponse  =
    list
        object7 User
            ("id"      := int)
            ("first"   := string)
            ("last"    := string)
            ("emails"  := list string)
````

We also need a little utility function for building the query URL from the search parameters:

````haskell
queryUrl : Search -> String
queryUrl search =
    url "https://www.schoolshape.com/fatcontroller/search" [
        ("first", search.first),
        ("last",  search.last),
        ("email", search.email),
        ("org",   search.org),
        ("type",  search.typ)
    ] 
````

Elm has a new `Task` interface to deal with asynchronous requests, which makes requesting server data easier. The idea is that our searches signal should be converted to a signal of Http fetch tasks, which in turn is sent out of a `port` in order to execute the search.

````haskell
makeHttpRequest : Search -> Task Error (List User)
makeHttpRequest search = getJson decodeResponse (queryUrl search) 

port httpRequests : Signal (Task Error (List User))
port httpRequests = Signal.map makeHttpRequest searches
````

This will now send http requests every time we hit enter in any of our fields, but will then throw away the result.  We need to expand makeHttpRequest to do something after the request completes:

````haskell
searchResults : Mailbox (Result Error (List User))
searchResults = mailbox (Ok [])


report : Result Error (List User) -> Task Result ()
report = Signal.send searchResults.address


makeHttpRequest : Search -> Task x ()
makeHttpRequest search =
    getJson decodeResponse (queryUrl search) `andThen` report
````

We created a mailbox to send our search results to, created a function 'report' that takes coded results and sends them to the mailbox, and then used `andThen` to run the report after every http response.  We can now get a signal of search results using `searchResults.signal`.


Finally, we need to display the list of fetched users (or an error).  This is thankfully straightforward using the html api.

````haskell
renderUser : User -> Html
renderUser user =
    let fields = [
            text <| toString user.id,
            text <| user.first,
            text <| user.last,
            div [] <| List.map renderEmail user.emails,
            ]
        renderEmail email = div [] [
                a [ href <| "mailto:%22" ++ user.first ++ " " ++ user.last ++ "%22%3c" ++ email ++ "%3e" ] [ text email ]
            ]
        makeTD field = td [] [field]
    in tr [class "row"] (List.map makeTD fields)


renderUsers : List User -> Html
renderUsers users = 
    let makeHeader fieldName = th [] [text fieldName]
        fieldNames = [
            "Id",
            "First name",
            "Surname",
            "Emails"
            ]
        header = tr [] <| List.map makeHeader fieldNames
    in  table [] <| header :: List.map renderUser users 


render : Result Error (List User) -> Html
render result =
    let resultDisplay = case result of 
            Ok users      -> renderUsers users
            Err errorMsg  -> div [] [text (toString errorMsg)]
    in  div [] [ inputs, resultDisplay ]


main : Signal Html
main = render <~ searchResults.signal
````


Conclusion
==========

Having completed the whole project in a long day's work, I would have to say that I could have done it in less thna half the time using Angular, React or some other javascript framework, even figuring for the extra time spent learning the new APIs.  Several aspects of the API feel cumbersome and low-level, especially dealing with streams of events from individual input fields and combining them appropriately - in an imperative language we'd simply go get all the values we wanted every time we received a submit event, and our work is done.  Also, the need to decode JSON values rather than use them directly as you can in a dynamic language seems unnecessarily time-consuming, especially as all the decoders will be need to be updated over the life of the application as the data requested from the server changes.  The language overall feels very rigid - every tiny error you make will cause a compilation failure, and must be corrected exactly before you can continue.  You have to think carefully about the type of every value, and even finding a way to tap into streams to get debugging output to the screen can be fiddly.  You MUST deal with any potential errors or missing values, and have to write at least some makeshift code to deal with them before being able to run the code to check the main use case.

However, that rigidity and fussiness of the language means that now the application is finished, I feel 100% certain that there are no bugs in the code.  I spent very little of my time debugging - as soon as the code compiled, it almost always did what I expected the first time.  The only debugging session I had to go through turned out to be incorrect CORS headers, nothing to do with my code at all.

These virtues will begin to shine through on larger apps, especially complex apps where it is difficult to separate and loosely couple components.   I would probably say that using Elm will pay off for apps larger than around 5000LOC.  Also, the purity and immutability of the language should make it easier to build high-level libraries to deal with many of the low-level details, which could bring Elm's sweet spot down to much smaller apps.

All in all I would say that Elm isn't yet the go-to tool for making standard small apps, but it does hold a lot of promise for the future, especially for more complex and non-standard apps that aren't served well by existing frameworks.

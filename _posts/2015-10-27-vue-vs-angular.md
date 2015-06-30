---
layout: post
title: Comparing Vue.js vs Angular.js
---


Although Vue already has a [page](http://vuejs.org/guide/comparison.html) comparing the two, I'd like to point out a few things that I love about Vue that had been irritating me in Angular.

Raw HTML Binding
================

Although this is possible in AngularJS using ng-bind, you must configure the security context correctly to avoid errors, which is a pain, especially if you're just binding HTML as part of a debugging or refactoring effort rather than permanently.  Vue allows you to bind html with triple brackets, and figure out for yourself if it's secure or not, much more convenient.

````html
<div>\{\{\{ raw_html \}\}\}</div>
````

Simple custom directives
========================

Writing real-life directives can get hairy, filling in functions with names such as `controller`, `link`, `compile`, etc, and knowing that the controller is initialized before child directives while the link is run afterwards requires understanding Angular's lifecycle.  Vue is straight-forward:

````javascript
Vue.directive('my-directive', {
  bind: function () {
    // do preparation work
    // e.g. add event listeners or expensive stuff
    // that needs to be run only once
  },
  update: function (newValue, oldValue) {
    // do something based on the updated value
    // this will also be called for the initial value
  },
  unbind: function () {
    // do clean up work
    // e.g. remove event listeners added in bind()
  }
})
````

Simple Transitions
==================

The angular model for transitions can be a little confusing, it requires more css classes (with complex names) to be defined than are strictly necessary in most cases:


````javascript
.repeated-item.ng-enter.ng-enter-active,
.repeated-item.ng-move.ng-move-active {
  opacity:1;
}
````

Vue sticks at requiring only three self-explanatory classes, and offers js extension points for coordinating more complex transitions.

````css
/* always present */
.expand-transition {
...
}

/* .expand-enter defines the starting state for entering */
/* .expand-leave defines the ending state for leaving */
.expand-enter, .expand-leave {
...
}
````

Directive arguments
===================

Whereas Angular has one directive for each event (`ng-click`, `ng-blur`, etc), which still doesn't cover all possible events even after improvements with every version of angular, there is one simple syntax in Vue which covers all events.

````html
<button v-on:click="greet">Greet</button>
````

This applies equally to `bind:` which neatly binds any attribute.

Vue also has directive modifiers, a nice shorthand which saves an if statements here and there

````html
<input v-on:keyup.13="submit">
````

Removing superfluous divs
=========================

A pet peeve of mine about Angular is that some directives introduce a superfluous html tag into the rendered markup (I'm looking at you `ng-repeat`). This might not seem like a big issue, but it can waste time when exploring the DOM in the browser dev tools, and trip up some 3rd party libraries.  Vue has the `template` directive which removes itself on compilation, a little like Angular directives with `replace: true` set.

````html
<ul>
  <template v-for="item in items">
    <li>\{\{ item.msg \}\}</li>
    <li class="divider"></li>
  </template>
</ul>
````

Nice class bindings
===================

The most common way that you will bind a class to your element is in a binary fashion; if some boolean in your model is true, the class is present, otherwise it's absent.  In Angular (at least in 1.4), this is mostly easily done with the rather arcane:

````html
<div ng-class="shouldClassBePresent && 'className'">
````

Vue has a nice syntax for this situation.

````html
<div v-bind:class="{ 'class-a': isA, 'class-b': isB }"></div>
````

There's also neat syntax for binding inline styles.


Of course, Vue isn't a full framework like Angular, you'll have to pick another library for things such as http requests, modularization, interacting with the browser, etc, but I personally like separating these things as much as possible so it's easy to swap out one component at a later date.  Vue generally feels slicker and simpler to work with than Angular, and has better performance (though I'm not sure how it compares to virtual-dom-based frameworks on this).  It's also lighter to download at 23k gzipped vs Angular which starts at 36k with just the bare framework and can hit 100k with all components and plugins included.


Now, if only there was a binding for Vue with a haskell-like language....

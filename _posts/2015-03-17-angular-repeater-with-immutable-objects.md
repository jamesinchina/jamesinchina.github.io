---
layout: post
title: How to use collections of immutable objects in an Angular repeater
---

One thing about Angular repeaters has been annoying me for some time.

As a matter of good coding habits, I often use `Object.freeze` to make immutable objects that shouldn't be modified in the future.  This is a central tenet of functional programming and has helped me kill off a number of potentially insidious bugs before they were written into the code.  I also live in hope that one day javascript VMs will take advantage of immutability and all the nice functional code I wrote will get faster.

However, I've found myself commenting out `Object.freeze` over and over again when using Angular.  Why?  Because Angular repeaters use hash keys to detect which objects in a collection changed, and write those hash keys onto the objects themselves - not very functional.  This means that passing an area of immutable objects to a repeater results in the following (in Chrome):

````
TypeError: Can't add property $$hashKey, object is not extensible 
(angular.js:9193)
````

Today, though I found a workaround to this issue that I'd like to share.  If instead of an array, you pass an object, angular will use the keys of the object as the hashkey for the repeater item.  If you item has a natural id, you can use that:

````coffeescript
# myCollection is a collection of immutable objects
$scope.myCollection = {}
$scope.myCollection[obj.id] = obj for obj in myCollection
````

But, any old key will do.

````coffeescript
# myCollection is a collection of immutable objects
hashKey = 0
$scope.myCollection = {}
$scope.myCollection[hashKey++] = obj for obj in myCollection
````

No more errors, and we can keep our immutable objects.

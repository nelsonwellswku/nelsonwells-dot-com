+++
title = 'Swap object keys and values in Javascript'
date = 2011-10-11T15:00:00-05:00
draft = false
tags = ['programming', 'javascript']
+++
There are times when I would like to do work with the values in a Javascript object used as a hash map.  For example, sometimes I'd like to iterate over the values and perform an action on them, or perhaps check that a value exists in the hash map.  In this post, I'll write a function that inverts the keys and values in a hash map, give a couple of examples in regards to how and why you'd want to do that, and explain the short-comings of the approach.

<!--more-->

## The Invert Function

    var invert = function (obj) {

      var new_obj = {};

      for (var prop in obj) {
        if(obj.hasOwnProperty(prop)) {
          new_obj[obj[prop]] = prop;
        }
      }

      return new_obj;
    };

The function itself is fairly simple.  We'll pass an object as a parameter; this is the object whose keys and values we'll invert.  We then create a new empty object; this is the object that we will return with the inverted keys and values.  We do not operate on the object that we pass as a parameter because we do not want to change the original object.

First, we'll loop through the properties of obj.  We'll use `hasOwnProperty` to verify that the keys we invert were defined on the object itself and not up the prototype chain.  In the loop, we'll create a new key from the value of the key in the original object.  We'll assign the value of that key the property we're currently looking at in our iteration.  That's all there is to inverting the keys and values!

Of course, we'll return the newly generated object.

## Use Cases

Okay, so we can invert k, v pairs.  What can we do with that, though?  How about iterating over them and performing some kind of action on it?

    var obj = {
      firstname : "nelson",
      middlename : "dane",
      lastname: "wells",
    };

    var name = '';

    for(var value in invert(obj)) {
      name += value + ' ';
    }

    console.log(name.trim());

Maybe we want to know if a value exists in the hash map?

    var obj = {
      firstname : "nelson",
      middlename : "dane",
      lastname: "wells",
    };

    if(invert(obj)['dane']) {
      //value exists in the hash
      console.log("Value exists");
    } else {
      console.log("Value does not exist");
    }

    if(invert(obj)['terry']) {
      console.log("Value exists");
    } else {
      console.log("Value does not exist");
    };

Of course, there's nothing preventing you from inverting the keys, saving the inversion in a var, and doing something with it later.  In fact, you may want to do that...

## Shortcomings

There are some obvious shortcomings with this function.  First, every time we call it, we're iterating over the object's properties.  Of course, we're not going to want to do this in a loop because of performance implications.  Instead, we would want to call the function once and assign the result to a variable, and operate on _that_ object.

Another shortcoming is the hash map must have only one to one relationships.  If a value exists for two keys, when the object is inverted, we'll lose a key in the process.  It should be trivial to add exception handling to the function or some other mechanism to handle this deficiency gracefully, so if anyone is interested in that and can't do it themselves, I'd be happy to help.

We'll also prefer non-nested objects.  An array, another object, or a function may not make a great key :)

+++
title = 'JSON.stringify with mapped variables'
date = 2012-02-06 15:00:00-05:00
draft = false
tags = ['programming', 'javascript', 'mirth connect']
+++
There are times when I want to create a data structure in Mirth, and output the contents of that data structure while debugging the code.  Luckily, `JSON.stringify()` is available not only in modern browsers but also in Mirth.

    var o = {
      mrn: '8675309',
      labs: [
        'blood bank',
        'microbiology',
        'toxicology'
      ]
    };

    console.log(JSON.stringify(o, null, '\t'));

However, there are any types of Mirth mapped variables, the JSON.stringify() method blows up on itself with a message similar to

    DETAILS:    Java class "[B" has no public instance field or method named "toJSON".

This is how we fix it.

<!--more-->

Consider the following code.  Assume that a variable has been mapped to mrn using Mirth's mapper transformer or a call to `channelMap.put()`.  This code explodes with the aforementioned error.

    var o = {
      mrn: $('mrn'),
      labs: [
        'blood bank',
        'microbiology',
          'toxicology'
      ]
    };

Oddly, if we call `logger.error($('mrn'))`, the mapped variable shows up correctly.  Our assumption is that `$('mrn')` would return a string if the mapped variable was in fact a string (as opposed to a javascript object, for example).  However, if we then call `logger.error(typeof $('mrn'))`, we get "object."  This should be our first clue that not all is what it seems.

Since I am not a Java programmer, I searched the internet trying to find what `[B` represents.  I've found that `[B` is some kind of a byte array in Java, and thus Mirth or maybe even Rhino implements a mapped variable as a byte array.  When `JSON.stringify()` is called on our data structure, `toJSON()` is called on each individual element in the structure... since the Java byte array `[B` does not implement `toJSON()`, an exception is thrown.

So, our goal is to convert the `[B` object into a javascript string since a javascript string does implement the `toJSON()` method.  My original instinct was to simply called `toString()` on the mapped variable, but that failed similarly.

I eventually found that explicitly constructing a javascript string with the new keyword would get the job done.  This is the working code for printing a data structure with a mapped variable.

    var o = {
      mrn: new String($('mrn')),
      labs: [
        'blood bank',
        'microbiology',
        'toxicology'
      ]
    };

    logger.error(JSON.stringify(o, null, '\t'));

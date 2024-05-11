+++
title = 'Two reasons your MarkLogic code is failing silently, part 2'
date = 2013-05-19T15:00:00-05:00
draft = false
tags = ['programming', 'functional programming', 'marklogic', 'xquery']
+++
Silent failures are a programmer's worst nightmare, and in a world where first class debuggers are few and far between, those silent failures are sure to drive us crazy.  In the first half of this article, we discussed [Two reasons your MarkLogic code is failing silently, part 1]({{< ref "/posts/2012/10/two-reasons-your-marklogic-code-is-failing-silently-part-1" >}}) silent failures in MarkLogic due to XML namespace issues.  XML namespace issues are going to crop up in any implementation of XQuery, though, so there are many resources to reference in addition to my article.

In the second (and last) part of the article, we'll discuss a feature specific to MarkLogic's XQuery implementation.  This feature is called [function mapping](http://docs.marklogic.com/guide/xquery/enhanced#id_55459).

<!--more-->

## What is function mapping?

Function mapping is syntactic sugar for passing a for expression that iterates over a sequence as a parameter to a function.  Let me demonstrate... given this function

    declare function local:add-two($int as xs:int)
    {
        $int + 2
    };

I can apply that common looping pattern like this

    local:add-two(for $int in (1 to 5))

The output of this snippet of code is the sequence (3 4 5 6 7).  Similarly, this snippet of code which uses function mapping has the same output.

    local:add-two((1 to 5))

The function mapping solution is shorter and appears easier to read at first glance, but is it really?  If we look at the signature of the add-two function, it accepts a single integer as a parameter, not a sequence of integers.  I think it is misleading, especially if you are unfamiliar with function mapping.

However, that's exactly what function mapping is.  **Function mapping is equivalent to passing a for expression as a parameter.**

## How does function mapping cause silent failures?

Function mapping causes failures for the same reason a for loop iterating over an empty sequence fails to call a function, there is nothing to iterate over!  If a sequence is passed to a function expecting a single value, function mapping will execute that function _n_ times, where _n_ is equal to the length of the sequence.  If _n_ is zero, the function doesn't execute even when you expect it to.  If the sequence you're iterating over is a computed sequence, or the function you're calling takes multiple parameters, it is not always obvious why the function is not being called.  Running this code in MarkLogic's query console should illustrate how function mapping can be misleading.  I would expect 3 person XML elements, the first with the alias "street fighter master", the second with the aliases "tekken master" and "mortal kombat master", and the third with no aliases.  However, that's not what we get at all!

    declare function local:reformat-person(
        $firstname as xs:string,
        $lastname as xs:string,
        $alias as xs:string)
    {
        <person>
            <firstname>{$firstname}</firstname>
            <lastname>{$lastname}</lastname>
            <alias>{$alias}</alias>
        </person>
    };

    let $firstname := "nelson"
    let $lastname := "wells"
    let $alias := "street fighter master"
    let $aliases := ("tekken master", "mortal kombat master")
    let $missing-aliases := ()
    return <persons>{(
        local:reformat($firstname, $lastname, $alias),
        local:reformat($firstname, $lastname, $aliases),
        local:reformat($firstname, $lastname, $missing-aliases)
    )}</persons>

returns

    <persons>
        <person>
            <firstname>nelson</firstname>
            <lastname>wells</lastname>
            <alias>street fighter master</alias>
        </person>
        <person>
            <firstname>nelson</firstname>
            <lastname>wells</lastname>
            <alias>tekken master</alias>
        </person>
        <person>
            <firstname>nelson</firstname>
            <lastname>wells</lastname>
            <alias>mortal kombat master</alias>
        </person>
    </persons>

The intent was that an alias was optional, so it is true that the signature of the method was wrong for the intent.  Unfortunately, as programmers, we are not always right 100% of the time.  In any case, try running the above code without the first two calls to reformat-person.  You'll get no output at all!

Debugging the code is significantly more complicated when the failure is silent.  Trust me when I say that this is a contrived example and debugging more complex code can be a pain because of function mapping!

## Turning off function mapping 

While function mapping is a novel concept, I think it is more trouble than it is worth.  I like conciseness in my code as much as the next person, but I prefer readability and clarity even moreso.  Luckily, toggling function mapping on and off is simple.  This needs to be done at a module level.

    declare option xdmp:mapping "false";

When you pass a sequence to a function that expects a singleton, you'll get an error message similar to this one

`[1.0-ml] XDMP-AS: (err:XPTY0004) $alias as xs:string -- Invalid coercion: ("tekken master", "mortal kombat master") as xs:string`

Alternatively, you can also declare your XQuery module to run as XQuery 1.0 instead of XQuery 1.0-ml, but I don't recommend this approach.  If you use declare XQuery version other than 1.0-ml, you lose some features that _are_ worth having that are proprietary to the MarkLogic engine, like exception handling and transaction support.

Function mapping can be a source of frustration when writing new XQuery code, but hopefully now you will be prepared to work around some of its awkwardness.

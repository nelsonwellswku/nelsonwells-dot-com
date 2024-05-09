+++
title = 'Replace Switch Statements With Dictionaries'
date = 2015-02-22T15:00:00-05:00
draft = true
tags = ['programming']
+++
I don't care for switch statements in C#. They're a poor substitution for [pattern matching](http://en.wikipedia.org/wiki/Pattern_matching). They require a compile time constant like a string, an enum, or an integral. Because of that restriction, they are not even a good replacement for giant if-else constructs, which was their intended purpose. On top of all of that, their syntax is ugly and error-prone. Ever left out a break; and saw some unintended consequences due to the fall through?

In general, I regard switch statements a code smell. Usually, some factoring could remove the need for them completely. However, there are times when you just have to make due with switch statements, like you're on a tight deadline so refactoring is out the question, or maybe there just isn't a good way to refactor it out of existence. If you're stuck with a switch statement, there may be a better solution. Let me show you how to turn ugly switch statements into something more manageable.

<!--more-->

## The switch statement

In two weeks, I've managed to write two switch statements, and both times I felt bad about it. One of them I was able to factor out of existence. The other, in [my Yahtzee game](https://github.com/nelsonwellswku/Yahtzee), looked _something like_ this. I'm going to return a string instead of an int because it is easier to demonstrate in the [demo application on Github](https://github.com/nelsonwellswku/blog-stuff).

    var score = "Zero!";
    switch(scoreCategory)
    {
        case ScoreCategory.ThreeOfAKind:
            // Here we'd process the dice values to see 
            // if it was a valid three of a kind or not.
            // Maybe something like diceProcessor.IsValid(diceValues);
            score = "ThreeOfAKind";
            break;
        case ScoreCategory.FourOfAKind:
            score = "FourOfAKind";
            break;
       case ScoreCategory.FullHouse:
            score = "FullHouse";
            break;
        case ScoreCategory.SmallStraight:
            score = "SmallStraight";
            break;
        case ScoreCategory.LargeStraight:
            score = "LargeStraight";
            break;
        case ScoreCategory.Yahtzee:
            score = "Yahtzee";
            break;
        default:
            // no-op, return the already assigned value
    }

    return score;

But this, this is awful for some the reasons I listed in the introduction to this article. Let's look a little closer and see what we can do about this.

First, let's notice that scoreCategory is an enum. Based on the value of the enum we pass in, every single case in the switch statement will assign a value to the score variable. Also, there are no fall-through mechanisms because each has a break; before the next case. How does knowing these simple facts help us? We can decompose the switch statement into something more palatable.

The enum certainly looks like a dictionary key to me. The contents of the cases look like they could be factored into a function that returns the score. It might not be obvious from this snippet since there's only a comment indicating such, but I want to process in some way a collection of integers for each case as well. That sounds like a parameter to the function to me.

So, now that we've decomposed the problem a little bit, let's look at the result.

## The dictionary

The switch statement can be replaced with this

    private static readonly ScoreDict ScoreDict = new ScoreDict
    {
        {ScoreCategory.ThreeOfAKind, values => {
            // Here we'd process the dice values to see 
            // if it was a valid three of a kind or not.
            // Maybe something like diceProcessor.IsValid(diceValues);
            return "ThreeOfAKind"}
        },
        {ScoreCategory.FourOfAKind, values => "FourOfAKind"},
        {ScoreCategory.FullHouse, values => "FullHouse"},
        {ScoreCategory.SmallStraight, values => "SmallStraight"},
        {ScoreCategory.LargeStraight, values => "LargeStraight"},
        {ScoreCategory.Yahtzee, values => "Yahtzee"}
    };

Much more succinct! Of course, it's just a dictionary and you may be wondering how to use it to accomplish the same goal as the switch statement. How does it replace the default case in the switch statement? There are just a couple more lines of code to bring it all together.

    Func<ICollection<int>, string> scoreFunc;
    return ScoreDict.TryGetValue(scoreCategory, out scoreFunc) ?
        scoreFunc(diceValues) :
        "Zero!";

First, we'll try to get the dictionary value, a function that takes a collection of ints and returns a string, and if it is in the dictionary, we'll execute it on the dice values we passed in. This mimics all of the case statements in the switch with the exception of the default. Instead, that is handled by the dictionary not even having whatever key we're trying to get. In that case, we'll return the default value of "Zero!" just like in our switch statements default case.

## Conclusion

It's true, there's probably a bit more overhead in the dictionary than in the switch statement. You have to store the dictionary somewhere so it's obviously using _some_ memory. However, in return for a little extra memory and function call overhead, you're getting maintainability and clean, concise code. This is one of those times where "premature optimization is the root of all evil" comes to mind.

As usual, the snippets of code used in this article are [available on Github](https://github.com/nelsonwellswku/blog-stuff). Additionally, you can see where I actually used this pattern in a real application in [this commit](https://github.com/nelsonwellswku/Yahtzee/commit/4baca0298e0ff6955e0248156eb295a2949bd046) to my Yahtzee game.

Happy coding!

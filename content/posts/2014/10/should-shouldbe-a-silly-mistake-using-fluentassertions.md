+++
title = 'Should, ShouldBe; a silly mistake using FluentAssertions'
date = 2014-10-14T15:00:00-05:00
draft = false
tags = ['programming', 'csharp', 'fluent assertions']
+++
I use [FluentAssertions](https://github.com/dennisdoomen/fluentassertions) for all of my asserting needs. I like it's API better than the assert methods you get with the .Net framework, and the FluentAssertions library provides an overall more fully featured set of assertion options. It's open-source and continually updated, too, which makes it all right in my book. Sometimes, though, while I'm furiously writing tests, I get this test failure signature and it catches me off guard.

`Subject has property Subject that the other object does not have.`

<!--more-->

Here's a sample NUnit test case that would cause this particular error.

    internal class MistakesTests
    {
        [Test]
        public void ThisDumbThingIKeepDoing()
        {
            var thing = new Thing { FirstName = "Nelson" };
            var thing2 = new Thing { FirstName = "Nelson" };

            // This should definitely pass, right?
            thing.Should().ShouldBeEquivalentTo(thing2);
        }

        private class Thing
        {
            public string FirstName { get; set; }
        }
    }

As you can see, I'm just doing a very simple object graph comparison, but it isn't working as I intended. Can you spot the error?

Whereas I should be calling ShouldBeEquivalentTo on my Thing instance, I'm actually calling it on an instance of ObjectAssertions type from the FluentAssertions library. Whoops! 😳

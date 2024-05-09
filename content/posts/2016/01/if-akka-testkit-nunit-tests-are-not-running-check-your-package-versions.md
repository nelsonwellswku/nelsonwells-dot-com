+++
title = 'If Akka Testkit Nunit Tests Are Not Running Check Your Package Versions'
date = 2016-01-18T15:00:00-05:00
draft = true
tags = ['programming', 'csharp', 'akka.net']
+++
As of this writing, Akka.TestUnit.NUnit and NUnit 3 are incompatible with one another. Specifically, package version 1.0.5 and NUnit 3.0.1. I've opened an issue with the Akka.net team [Issue 1651](https://github.com/akkadotnet/akka.net/issues/1651) so you can watch it for the resolution to this problem.

I was trying to test Akka.net actors in a project that was already using NUnit 3.x. I wrote the tests, but when I ran them with the ReSharper test runner, they would show up for just a split-second and then disappear. I thought that maybe it was the ReSharper test runner since that very day I had to upgrade from R# 9 to R# 10 to get the NUnit 3 support. I tried running the tests from the NUnit 3 console runner and experienced the same thing, though. The tests weren't failing or marked ignored, or even "not run", the test runner just didn't pick them up at all. With a little help from the good folks at the Akka.net Gitter chat, I was able to get my tests running by downgrading to NUnit 2.6. I suppose it is a "known" issue but maybe not that well known since there was no issue logged for it :)

<!--more-->

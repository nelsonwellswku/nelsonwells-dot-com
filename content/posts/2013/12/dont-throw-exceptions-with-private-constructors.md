+++
title = "Don't throw exceptions with internal constructors"
date = 2013-12-01T15:00:00
draft = false
tags = ['programming', 'csharp', 'api design']
+++
I ran into an annoying problem the other day when working with a 3rd party library, [mail.dll](http://www.limilabs.com/mail).  You see, mail.dll exposes library methods for sending email messages to an SMTP server, functionality common enough that you should never have to write it yourself.  While I've been pleased with its ability to perform SMTP conversations, I have not been impressed with its use of exceptions.

When the mail.dll SMTP library fails to connect or get a response from the SMTP server to which it is connecting, it throws a mail.dll-specific custom exception, SmtpResponseException.  Likewise for the IMAP and POP3 libraries, if a failure happens during client interaction with the server, an ImapResponseException or Pop3ResponseException is thrown, respectively.  Since, in the normal course of a properly configured mail client's operation, the expectation is that the server will respond appropriately, I don't have a problem with these libraries throwing exceptions.  They are understood and can be handled by the consuming application.

Here's the annoying bit. The custom exceptions thrown by mail.dll only have internal constructors and cannot be thrown outside of the mail.dll assembly.  In this article, I'll talk about why your library should not throw exceptions that the consuming code cannot throw itself and a workaround when someone else's library does.

<!--more-->

## The problem

There are two things that you should know before I go any further describing the problem. First, when handling exceptions, a best practice is to only catch the most specific exception that you can handle for a given code block. As a quick example, you don't want to catch the generic Exception when the only exception you can capably handle is a FileNotFoundException.  If the code you're wrapping in a try block runs out of memory and throws an OutOfMemoryException, whatever you were trying to do to that would handle a FileNotFoundException will probably fall flat on its face.

Secondly, I believe that code written should be as tested as feasible through unit tests. This is really the crux of the problem. If you write an API that throws an exception that only has a private constructor, consuming code cannot setup a mock of your class to throw the exception.

## The third party library

I feel like a (contrived) code sample will do better justice to the problem at hand than my prose. Consider these code files and imagine them as the third party library, in its own assembly, that you want to include as a dependency in your project.

    // IAdder.cs
    namespace Adder
    {
        public interface IAdder
        {
            int Add(int x, int y);
        }
    }

    // WholeNumberAdder.cs
    namespace Adder
    {
        public class WholeNumberAdder : IAdder
        {
            public int Add(int x, int y)
            {
                if (x < 0) throw new WholeAdderException("Argument x less than 0");
                if (y < 0) throw new WholeAdderException("Argument y less than 0");

                return x + y;
            }
        }
    }

Notice that WholeNumberAdder will throw a WholeNumberAdderException in the Add method if either operand is less than 0. This error cases that cause exceptions to be thrown will hopefully be in the documentation to the third party library you're consuming, and you should feel comfortable enough to mock the error case and exception instead of trying to re-create it. We'll see more about this later.  For now, let's look at the implementation of WholeAdderException.  Remember, this is **not what you want to do!

    // WholeAdderException.cs
    using System;

    namespace Adder
    {
        public class WholeNumberAdderException : Exception
        {
            internal WholeNumberAdderException() : base() { }
            internal WholeNumberAdderException(string message) : base(message) { }
            internal WholeNumberAdderException(string message, Exception innerException) : base(message, innerException) { }
        }
    }

Here, we can see that all of the constructors are marked [internal](http://msdn.microsoft.com/en-us/library/7c5ka91b.aspx). Since your application will probably be consuming the 3rd party library in its own assembly, there's no way for you to construct one of these WholeNumberAdderException types.

## How internal constructors break unit testing

Suppose we're creating a class used for doing mathematical operations. Let's call it MathMachine.  One of the operations we want to support is adding two numbers; any math machine should be able to do that.  We're lucky because we already know of a 3rd party library to handle the complexity of adding two numbers.  We'll take it on as a dependency to our MathMachine and save ourselves some time developing our own adder type.  Since we practice TDD, we'll write the positive unit test for our new MathMachine first.

    [TestClass]
    public class MathMachineTests
    {
        private Mock<IAdder> _adderMock;

        public MathMachineTests()
        {
            _adderMock = new Mock<IAdder>();
        }

        [TestMethod]
        public void Add_Success()
        {
            // Arrange
            var expected = 5;
            _adderMock.Setup(x => x.Add(2, 3)).Returns(expected);
            var machine = new MathMachine(_adderMock.Object);

            // Act
            var actual = machine.Add(2, 3);

            // Assert
            actual.Should().Be(expected);
            }
        }
    }

So, we have a unit test for our math machine's addition functionality.  We see that our MathMachine will take a dependency on an IAdder instance provided via constructor parameter.  Let's write our MathMachine.

    public class MathMachine
    {
        private readonly IAdder _adder;

        public MathMachine(IAdder adder)
        {
            _adder = adder;
        }

        public int Add(int x, int y)
        {
            return _adder.Add(x, y);         
        }
    }

So far, so good.  We have a math machine that can add two numbers.  However, we know that the concrete implementation we're going to use will only add whole numbers.  However, in our MathMachine, we want to handle a negative number parameter by converting it to a positive number.  For instance, in our MathMachine, Add(-3, 2) should yield 5.  Let's write a unit test for this case, knowing that the IAdder interface that we intend to use will throw a WholeAdderException in the case of a a negative parameter.

    [TestMethod]
    public void Add_Failure()
    {
        // Arrange
        var expected = 5;
        _adderMock.Setup(x => x.Add(3, -2)).Throws(new WholeNumberAdderException(It.IsAny<string>()));
        var machine = new MathMachine(_adderMock.Object);

        // Act
        var actual = machine.Add(3, -2);

        // Assert
        actual.Should().Be(expected);
    }

We've set up our mock IAdder to throw a WholeAdderException when passed 3, -2 to the Add method.  We've called Add on our MathMachine and then asserted that the result should be 5 and the exception thrown by the IAdder should be swallowed.  All we have to do is hit compile and... crap!  The unit test doesn't even compile.  Why?

`'Adder.WholeNumberAdderException' does not contain a constructor that takes 1 arguments`

If we look at the WholeNumberAdderException, we see that it definitely does have a constructor that accepts one argument.  But wait, **all of the constructors are internal.**  Since we cannot instantiate an instance of a WholeNumberAdderException from outside of the 3rd party assembly, we cannot set up our mock instance to throw it!  How aggravating!  Now, the only way to unit test our MathMachine Add method is to construct an instance of the concrete type and pass it as a dependency to our MathMachine.

    [TestMethod]
    public void Add_ConcreteAdderImplementation_Failure()
    {
        // Arrange
        // Since I can't throw the whole adder exception from my mock,
        // I have to use an instance of the concrete implementation. Yuck.
        var adder = new WholeNumberAdder();
        var expected = 5;
        var machine = new MathMachine(adder);

        // Act
        var actual = machine.Add(3, -2);

        // Assert
        actual.Should().Be(expected);
    }

Since we've instantiated a WholeNumberAdder and passed it as a dependency to the MathMachine instance, we can see by way of our action and assertion that the MathMachine is working correctly.  Adding 3 and -2 yields 5.

However, our unit test is not quite as pure as we'd like it to be.  Instead of mocking our dependencies, we've had to create them.  Imagine if the dependency we're creating were stateful (as opposed to stateless in the case of our WholeNumberAdder).  We could have to do many complex operations to coerce the dependency into a state where it would even throw an exception.  By that point, we're not only testing our MathMachine, but we're testing the internals / implementation details of an interface. Do we even really have a unit test at that point?

## Code and conclusions

Unfortunately, I don't know of any good workaround when dealing with internal or private exception type constructors in 3rd party libraries.  In my case, with mail.dll, there is now a chunk of code that cannot be unit tested because there is no way to coerce mail.dll into throwing the exception I'm looking to handle without adding externalities.  Naturally, I do not want those externalities polluting my unit tests.  Instead, we had to rely on integration tests.  Had mail.dll's API had been better thought out, we could have used unit tests which would have made me feel a little more confident in that particular piece of code.

Hopefully you've found that the take-away from this article is that, as a .Net developer, you should not be throwing exceptions that the consuming code cannot throw itself.

You can grab [the code from this article on Github](https://github.com/nelsonwellswku/blog-stuff).

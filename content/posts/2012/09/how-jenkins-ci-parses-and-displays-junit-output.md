+++
title = 'How Jenkins CI parses and displays jUnit output'
date = 2012-09-01T15:00:00-05:00
draft = false
tags = ['devops', 'jenkins', 'unit testing', 'xml', 'image']
+++
Now that I've had the opportunity to work on a continuous integration infrastructure for both Mirth Connect interfaces and a MarkLogic-powered REST API.  The Mirth interface testing software was built with Perl and TAP (Test Anything Protocol).  The ML REST API testing software was built on MarkLogic itself.  What they both had in common is that [Jenkins](http://jenkins-ci.org) was used to build the environments and run the tests.

I've really come to appreciate how useful Jenkins can be.  Jenkins can aggregate jUnit style reports out of the box, and Jenkins can give you meaningful textual and graphical representations of those reports.  It will tell you how many tests failed, how many tests passed, what class and packages were being tested, and it give you this information over the course of many builds.

However, since neither CI testing software was actually built on Java, I was unable to leverage the actual [jUnit testing framework](http://junit.org).  In addition, neither the Mirth Connect interfaces nor MarkLogic/Xquery have a concept of grouping code into packages and classes as a Java programmer may expect. Because of these two issues, I had to generate jUnit output programmatically.  Even though I was able to start with someones stab at the [jUnit XML schema](http://windyroad.org/dl/Open%20Source/JUnit.xsd), I found that Jenkins does not parse and display the jUnit output exactly as I would have expected.  For the time being, it is probably easier to ignore the jUnit schema and instead listen to what I'm about to say :)

<!--more-->

Before I go into detail, I want to be forthright about a couple of things.  Of everything I have learned and am about to write about, I have learned through my own experimentation.  I am writing about it because I could find no good source detailing how Jenkins parses jUnit output.  If there is a help page or doc that I have missed, please let me know!  Because of that, it is very possible and likely that Jenkins can use even more information from a jUnit report that I haven't discovered, so don't take this as Jenkins gospel.  Without further ado, let's look at an example output file that I dreamed up.  It may not be real-world accurate, but it will demonstrate what I want to show.

    <testsuites>
        <testsuite name="Customer Demographics Tests" tests="9" failures="0" timestamp="2012-08-23T14:40:44.874443-05:00">
            <testcase name="Insert First Name" classname="CustomerDemographicsPKG.Name"/>
            <testcase name="Insert Last Name" classname="CustomerDemographicsPKG.Name"/>
            <testcase name="Change First Name" classname="CustomerDemographicsPKG.Name"/>
            <testcase name="Change Last Name" classname="CustomerDemographicsPKG.Name"/>
            <testcase name="Delete First Name" classname="CustomerDemographicsPKG.Name"/>
            <testcase name="Delete Last Name" classname="CustomerDemographicsPKG.Name"/>
            <testcase name="Insert DOB" classname="CustomerDemographicsPKG.Misc"/>
            <testcase name="Update DOB" classname="CustomerDemographicsPKG.Misc"/>
            <testcase name="Delete DOB" classname="CustomerDemographicsPKG.Misc"/>
        </testsuite>
        <testsuite name="Customer Contact Tests" tests="4" failures="1" timestamp="2012-08-23T14:40:44.874443-05:00">
            <testcase name="Insert Street" classname="CustomerContact.Address"/>
            <testcase name="Insert City" classname="CustomerContact.Address"/>
            <testcase name="Insert State" classname="CustomerContact.Address"/>
            <testcase name="Zip" classname="CustomerContact.Address">
                <error message="Zip code out of range">Zip code 139819 is not in range of valid zip codes.</error>
            </testcase>
            <testcase name="Insert Home Phone" classname="CustomerContact.Phone"/>
            <testcase name="Insert Cell Phone" classname="CustomerContact.Phone"/>
            <testcase name="Insert Business Phone" classname="CustomerContact.Phone"/>
        </testsuite>
    </testsuites>

## Main project status page
Even though the main project status page doesn't directly render anything from the jUnit report, I want to show it anyway.  In particular, the project status page gives us a list of build history in the left pane and some permalinks to important build events such as last stable, last failed, last unsuccessful build, and a few others.  One of my favorite features is the graph which plots number of tests performed and number of failures over an arbitrary number of builds.  Very cool.

![Jenkins jUnit project home](/images/jenkins-junit-project-home.jpg)

As you can see, there is a link for "Latest Test Result", and that link indicates that there was 1 failure.  Since none of the generated report is used on this page, go ahead and click the "Latest Test Result" link and we'll look at...

## Package level test results
Finally, the good stuff!  As an aside, I want to point out that this page was my inspiration for writing this post.  First, in my most recent work, Xquery does have any understanding of packages; it is not OOP, it is functional, and does not support grouping modules as "packages."  Secondly, the "package" attribute on the "TestSuite" element is not parsed by Jenkins at all!  It took a minute or six of Googling to narrow down how to get Jenkins to separate my tests into packages instead of grouping them all as "root."

![Jenkins jUnit package list](/images/jenkins-junit-package-list.jpg)

First, we'll see a list of all the individual tests that failed, regardless of which package that test belonged to.  Then, we'll see a list of packages that were tested for this build.

For the "All Failed Tests" list, you can get a preview of what we'll be discussing shortly.  Jenkins identifies the failed tests as {package}.{class}.{test}.  Similarly, the packages listed in the "All Tests" list are parsed from the jUnit report based the "classname" attribute on the "testcase" elements where the left hand of the period is the package name and the right hand of the period is the class name.  I think that is misleading.  In any case, by examining this snippet and the above image, I hope it is easy to visualize what I just explained :).

    <!-- All the information on this page is based on the testcase element -->
    <testcase name="Insert First Name" classname="CustomerDemographicsPKG.Name"/>

Next, we'll dive a page deeper by clicking a package name.  Since CustomerContact package has a failure, I will examine that one.  That link leads us to...

## Class level test results

This page is very similar to the previous page, except instead of listing tests at the package level, they're listed at the class level.  We still do get a list of all the tests that failed.

![Jenkins jUnit class list](/images/jenkins-junit-class-list.jpg)

I think this page is even easier to see how the the XML is visualized.  Notice that there are a total of 7 tests, 4 from the Address class and 3 from the Phone class, and as mentioned earlier, Address and Phone are on the right hand side of the period in the "classname" attribute of the "testcase" elements.

    <testcase name="Insert Street" classname="CustomerContact.Address"/>
    <testcase name="Insert City" classname="CustomerContact.Address"/>
    <testcase name="Insert State" classname="CustomerContact.Address"/>
    <testcase name="Zip" classname="CustomerContact.Address">
      <error message="Zip code out of range">Zip code 139819 is not in range of valid zip codes.</error>
    </testcase>
    <testcase name="Insert Home Phone" classname="CustomerContact.Phone"/>
    <testcase name="Insert Cell Phone" classname="CustomerContact.Phone"/>
    <testcase name="Insert Business Phone" classname="CustomerContact.Phone"/>

Now is a good time to notice that, under the "All Failed Tests", the test name really, truly is parsed from the "name" attribute on the "testcase" elements.  The "classname" attribute compounds {package}.{class} and "name" visualizes the name of the test.  We'll see that more obviously on the next page, after I click "Address" in the list of classes tested.

## The results of the actual tests!

This page is less exciting, but it arguably the most important.  Here, we get a clean, simple table of the tests that were executed and whether they passed or failed.  Also, depending on the same test on the previous build, the test's status may also be regression or fixed.

![Jenkins jUnit test list](/images/jenkins-junit-test-list.jpg)

It's easy to see how the test names are generated from the jUnit reports.  They are simply the "name" attribute on the "testcase" elements.

    <testcase name="Insert Street" classname="CustomerContact.Address"/>
    <testcase name="Insert City" classname="CustomerContact.Address"/>
    <testcase name="Insert State" classname="CustomerContact.Address"/>
    <testcase name="Zip" classname="CustomerContact.Address">
        <error message="Zip code out of range">Zip code 139819 is not in range of valid zip codes.</error>
    </testcase>

Since there were three tests that passed, and those tests aren't going to give any interesting output, we're going to look at the failed test, Zip, and see what we get.

## Oh no!  A failed test

I said in the previous section that the list of tests that passed and failed was arguably the most important page.  I said arguably because it is _this_ page, the failed test page, that will give us some good information about how to address whatever bug or bad data caused the test to fail.

![Jenkins jUnit failed test output](/images/jenkins-junit-failed-test-output.jpg)

On this page, the Error Message is drawn from the "error" attribute on the "message" element (who would have guessed?).  This is typically a one line explanation of what went wrong.  Then, the stack trace is the **value** of the "message" element.  There is no length restriction that I know of on this value, and it can contain any valid XML characters.  In my mockup, it is just a one liner, but hopefully whatever test harness is producing the output will give a real stacktrace if it exists.

    <testcase name="Zip" classname="CustomerContact.Address">
        <error message="Zip code out of range">Zip code 139819 is not in range of valid zip codes.</error>
    </testcase>

Also, you should notice From Customer Contact Tests in parenthesis at the top of this page.  Interestingly enough, even though that string is defined in the "testsuite" element higher in the hierarchy, it only shows up on this page!

    <testsuite name="Customer Contact Tests" tests="4" failures="1" timestamp="2012-08-23T14:40:44.874443-05:00">

Yet another thing to notice is that even though I have the tests and failures attributes in the "testsuite" element, Jenkins is smarter than displaying those values where those items are actually displayed on the page.  Instead, Jenkins keeps track of how many "testcase" and "error" elements exist in the collection of tests.  Pretty cool.

## Conclusion

Jenkins is a really useful piece of software.  However, if you're not using a testing framework that generates valid jUnit output for Jenkins to consume, or you are building your home testing framework or harness, it can be frustrating trying to figure outhow Jenkins parses jUnit XML.  Hopefully I have saved some of you the trouble of fudging around with jUnit and feeding it to Jenkins to see how it is parsed, or at least gave you a good head start.

As I become even more familiar myself with Jenkins, I'd like to elaborate on this entry as well as do some others.  Jenkins is very powerful, and it is a handy tool in any programmers belt.

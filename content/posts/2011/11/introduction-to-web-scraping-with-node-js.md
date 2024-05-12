+++
title = 'Introduction to web scraping with Node.js'
date = 2011-11-08T15:00:00-05:00
draft = false
tags = ['programming', 'javascript', 'nodejs', 'regex']
+++
The internet has a wealth of freely available information in difficult to consume formats.  For example, [ClicheSite](http://www.clichesite.com) has lists of cliches, euphemisms, and other phrases that are just _perfect_ for a hangman or Wheel of Fortune game.  Coincidentally, I wanted to write such a game to learn some new technologies, so I wrote a short, simple Node.js script to scrape some pages for those phrases. 

In this tutorial, I'll discuss how to make an HTTP request and get a page's HTML, how to parse the HTML for specific information, and why you would and would not want to do this in the first place.

<!--more-->

## Identifying Necessary Information

ClicheSite obviously does not have all of it's cliches listed on the home page.  It is technically feasible to begin at the homepage and follow links until we find something useful, but sometimes it makes more sense to take a manual step or two.  In this case, I identified alphabetic listings of phrases; each letter in the alphabet has its own page.  For example, `http://clichesite.com/alpha_list.asp?which=lett+1` is a list of all phrases beginning with the letter A and `http://clichesite.com/alpha_list.asp?which=lett+26` is a list of all phrases beginning with the letter Z.  So, it is clear that we can open and parse 26 pages to grab our phrases.

Next, we need to look at the pages' HTML source and identify a way to grab the data we want.  I used Firebug to look at the HTML, but you can also just look at the source directly with your favorite browser.  In Firefox, for example, you can right click anywhere on the page and hit "View page source."  In any case, here is a portion of the HTML we're concerned with.

    <tr>
        <td valign="top" bgcolor="#FFFFCC">&amp;nbsp;</td>
        <td valign="top" bgcolor="#FFFFCC">
            <span> <a href="content.asp?which=tip+1332">Acorn doesn't fall far from the tree, The</a></span>
        </td>
        <td valign="top" bgcolor="#FFFFCC">
            <span> United States</span>
        </td>
    </tr>

    <tr>
        <td valign="top" bgcolor="#FFFFCC">&amp;nbsp;</td>
        <td valign="top" bgcolor="#FFFFEE">  
            <span> <a href="content.asp?which=tip+65">Actions speak louder than words</a></span>
        </td>
        <td valign="top" bgcolor="#FFFFEE">
            <span> United States</span>
        </td>
    </tr>

Let's examine what we have here.  The HTML shows us that we have two rows with 3 columns.  We see that the two phrases are in the second column wrapped in a span and anchored (linked).  One possible action we could take is to write an HTML parser or use a 3rd party library to parse the HTML and then drill down to that HTML node.  However, for this particular case, that solution is a shovel when all you really need is a spoon.  Instead, we'll use a regular expression to grab the phrases... but how?

Notice that each phrase is surrounded by an anchor tag, and inside that anchor tag is the word "tip".  That will be the basis for our regular expression.  The content between the anchor tag with "tip" in the href attribute will be our phrase.  We'll see that regular expression in the next section.

## The Code

Since the code is so short, I'll let you look over it and familiarize yourself with it before I explain it.

    var http = require('http');

    var options;
    var regex = /<a.*tip.*>(.*)<\/a>/g;

    for (var i = 1; i <= 26; i++) {

    options = {
        host: 'clichesite.com',
        path: '/alpha_list.asp?which=lett+' + i
    };

    http.get(options, function (res) {

        // accessible to 'data' and 'end' events
        var data = '';

        res.on('data', function (chunk) {
        data += chunk.toString();
        })
        .on('end', function () {
            while ((match = regex.exec(data))) {
            console.log(match[1]);
            }
        });
    });
    }

First, we load the HTTP module.  We'll use this to make HTTP requests to ClicheSite.  Then, we'll define an options variable.  This is used as a parameter in the HTTP request and identifies the host and path (effectively, the URL) to hit.  Finally, we define a regex literal to identify the phrases.  Let's break it apart piecewise.

```
<a   - less than followed by the letter a
.*   - followed by any character any number of times
tip  - followed by the letters t, i, p
.*   - followed by any character any number of times
>    - followed by a greater than
(.*) - followed by any character any number of times, and capture it
<    - followed by a less than
\/a  - followed by a forward slash and the letter a
>    - followed, finally, by greater than
```

We'll match it globally.  Later, we'll loop through the result of the regular expression and grab the captured phrase.

Now, since we've previously identified that we need to scrape pages where lett+ a number 1 through 26 appears in the URL, we'll loop through that range.  We'll populate that options variable we created earlier with an object with two properties, host and path.  The host is the domain we wish to hit with our HTTP request, and the path is basically everything past that point in the URL.  Notice that we're appending our loop index to the end of the path.  That is how we'll hit pages A through Z.

Now, we'll make an HTTP request.  All we have to do is call the get method of the HTTP object. We'll define a callback function with an HTTP result object as a parameter... we'll register events on this res object to do the actual scraping.  In the .get callback, we define an empty string called data.  This will eventually hold the full HTML content of the page.

Next, we register two events on the res object.  The first event, "data", is fired when data is received.  When that happens, we'll just append that data to the data string we defined.  The next event, "end" is fired when the HTTP request has finished.  When that happens, we will have the full HTML document.  This is where we do the work.

In the callback to this event, we loop through the results of our regular expression with the following construct:

    while((match = regex.exec(data)))

The regular expression exec will eventually evaluate to null, a falsey value, and the loop will end.  In the body of the loop, we can access the phrase with `match[1]`.  `match[0]`, of course, contains the whole matched expression; we just want the capturing group.  In this example, I simply print the phrase to the screen... you'll likely want to do more to the data than that, though.

## Pitfalls
One thing you'll notice about this solution is that the results are not in alphabetical order even though we iterate alphabetically from A to Z.  This is because of Node.js' event-driven, asynchronous nature.  In the body of the most broad loop, we make an HTTP request.  However, that request will most likely not finish before the we hit the next iteration of the loop.  Processing seed is much faster than network latency, after all.  In fact, when I was testing the script, I was often making 5-10 HTTP requests before the first one returned!  For this simple example, if you wanted a sorted list you could run the script and pipe to sort

    node index.js | sort

More likely, you'll want to store the results in a data structure and sort it or in a database so it can be sorted later.  This depends on context, of course.

Ironically, you also need to be wary of scraping pages heavy with Javascript.  If the contents of the page are dynamic, you may not have access to the data you actually want to get!

## Final Thoughts
Why scrape web pages?  If you are feeling black hat, it is relatively simple to scrape for email addresses, phone numbers, and addresses.  In the real world, you may want to scrape your own web applications to ensure data validity and correct formatting of data.

In any case, please be nice!  Don't break a website's terms of use, and most certainly do not blame it on me! :)

Happy coding

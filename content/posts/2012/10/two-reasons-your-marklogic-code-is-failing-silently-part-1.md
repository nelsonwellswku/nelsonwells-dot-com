+++
title = 'Two reasons your MarkLogic code is failing silently, part 1'
date = 2012-10-24T15:00:00-05:00
draft = false
tags = ['programming', 'marklogic', 'xquery']
+++
If you're new to MarkLogic application programming, it's easy to get frustrated when your code fails without any obvious reason why.  There's no run-time error, no exception to catch, and when you surround the code block in question with log writing expressions, the "before" log expression and the "after" log expression both work perfectly.

Even now, excellent application developers are still making these same mistakes because they're so easy to do if you aren't paying perfect attention.

**Coworker**: Hey, can you look at this code and see if you know what's wrong with it?

**Me:** _(before he pastes the code into the IM window)_

**Me**: namespaces or function mapping

**Coworker**: ...

**Coworker**: Damnit.

<!--more-->

## Namespace resolution

Here is an example of code that fails silently.  Can you see why?

    let $xml := <name xmlns="https://nelsonwells.com">
                  <firstname>Nelson</firstname>
                  <middlename>Dane</middlename>
                  <lastname>Wells</lastname>
                </name>

    return text{fn:concat(
      "First name: ", $xml/firstname/text()
    )}

This seems innocent enough.  You want the text of the <firstname> node that is a child of your root element.  However, **because your XML snippet is in the non-default namespace**, the Xpath expression will not resolve the way you expect.

### Qualify your XPath expressions

The first solution is to explicitly qualify the aliased namespace in your Xpath.

    (: namespace declarations go in the prolog :)
    declare namespace ndw = "https://nelsonwells.com";

    let $xml := <name xmlns="https://nelsonwells.com">
                 <firstname>Nelson</firstname>
                 <middlename>Dane</middlename>
                 <lastname>Wells</lastname>
                </name>

    return text{fn:concat(
      "First name: ", $xml/ndw:firstname/text()
    )}

It's important prefix all Xpath terms with the aliased namespace, not just one or the other.  If this XML snippet actually had a hierarchy, it may look like this instead

    $xml/ndw:addresses/ndw:billing-address/ndw:street/text()

This is usually my preferred solution, especially if you are dealing with multiple namespaces.

### Set the default namespace

You can also set the default namespace for a particular XQuery script in the prolog.  When you do this, all XPath expressions will be evaluated in that namespace unless you explicitly say otherwise.  Consider this block of code which will work the way you expect it to.

    xquery version "1.0-ml";
    declare default element namespace "https://nelsonwells.com";

    let $xml := <name xmlns="https://nelsonwells.com">
                  <firstname>Nelson</firstname>
                  <middlename>Dane</middlename>
                  <lastname>Wells</lastname>
                </name>

    return text{fn:concat(
      "First name: ", $xml/firstname/text(), "&amp;#xa;",
      "Middle name: ", $xml/middlename/text(), "&amp;#xa;", 
      "Last name: ",  $xml/lastname/text()
    )}

Here, we override the default namespace instead of letting it default to an empty namespace.  All XQuery expressions will be evaluated in the context of the declared default namespace.  Keep in mind that when you do this, if you are working with _any_ XML that isn't in the default namespace, you'll have to fall back to qualifying each XPath term as in the first example.  However, there is another way...

### Use xdmp:with-namespaces (MarkLogic specific)
The third option is to use a MarkLogic specific function, `xdmp:with-namespaces`.  `xdmp:with-namespaces` takes two parameters, one a sequence of prefix-namespace bindings and the second an XML element to evaluate.  Consider this example.

    xquery version "1.0-ml";

    let $xml := <name xmlns="https://nelsonwells.com">
                  <firstname>Nelson</firstname>
                  <middlename>Dane</middlename>
                  <lastname>Wells</lastname>
                </name>

    return text{ fn:concat(
        "Last name: ", 
        xdmp:with-namespaces( 
          ("", "https://nelsonwells.com"),
          $xml/lastname/text()
        )
      )
    }

Here, we've told `xdmp:with-namespaces` to evaluate the XPath expression passed as a second parameter such that no prefix should evaluate in the https://nelsonwells.com namespace. `xmpd:with-namespaces` allows you to easily work with XML in different namespaces.  Something to notice is that you don't even have to declare a namespace at all in the script's prolog as in the second example.

## Conclusion

You'll likely learn these methods your first day working with XQuery.  You won't get very far without them :).  However, I think it is important to have these gotchas all lumped together.  If it is 6PM and you're ready to call it quits for the day but you need a script working that is failing without any obvious reason, there's a good change it is a namespace issue.

There is also a good possibility that function mapping is causing you problems, too.  Come back soon for a new article about function mapping: learn what it is and how to avoid the pitfalls associated with it.  In the mean time, happy coding!

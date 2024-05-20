+++
title = 'jQuery change event on checkbox'
date = 2010-06-04T15:00:00-05:00
draft = false
tags = ['programming', 'javascript', 'jquery']
+++
A lot of times, you'll want to call a javascript function when a checkbox is changed from unchecked to check and vice versa.  Up until the release of jQuery 1.4, the obvious way to do it (the jQuery .change() event) did not work properly across all versions of Internet Explorer.  Instead, you had to check for the .click() event.  This was an accessibility problem because it did not fire when a user changed a checkbox with the space bar instead of by clicking.  Well, now we can rejoice.  The following code snippet works exactly how you think it would across browsers, including IE.

<!--more-->

    <!-- Load jQuery, and the necessary html -->
    <script type='text/javascript' src='http://ajax.googleapis.com/ajax/libs/jquery/1.4.2/jquery.min.js'></script>

    <input type='checkbox' name='my_checkbox' value='true' checked='checked'>

    $(document).ready(function(){
        $(":checkbox").change(function(){
            if($(this).attr("checked"))
            {
                //call the function to be fired
                //when your box changes from
                //unchecked to checked
            }
            else
            {
                //call the function to be fired
                //when your box changes from
                //checked to unchecked
            }
        });
    });

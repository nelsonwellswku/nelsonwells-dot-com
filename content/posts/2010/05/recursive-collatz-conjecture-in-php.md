+++
title = 'Recursive Collatz conjecture in PHP'
date = 2010-05-18T15:00:00-05:00
draft = false
tags = ['programming', 'php', 'recursion', 'math']
+++
I stumbled across a blog post by [Brie Gordon](http://unixsysadmin.org) about the Collatz conjecture, so naturally I had to Wikipedia it.  The simple story is that for any given integer, if it is even you divide it by 2 and if it is odd you multiply it by 3 and add 1. You continue this pattern with the returned numbers, and eventually the number will reach 1. So, to kill some time, I wrote a recursive version in PHP.   Here it is:

    function collatz($num)
    {   
        if($num == 1)
        {
            echo "<p style='color: red'>" . $num . "</p>\n";
            return;
        }

        if($num % 2 == 0) //even
        {
            echo "<p style='color: green'>" . $num . "</p>\n";
            return collatz($num / 2);
        }
        else
        {
            echo "<p style='color: red'>" . $num . "</p>\n";
            return collatz(3 * $num + 1);
        }
    }

<!--more-->

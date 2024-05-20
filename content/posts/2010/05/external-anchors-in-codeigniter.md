+++
title = 'External anchors in CodeIgniter'
date = 2010-05-07T15:00:00-05:00
draft = false
tags = ['programming', 'php', 'codeigniter']
+++
When using CodeIgniter, I almost never use bare anchor elements to create links.  Instead, I use the `anchor("controller/function", "text")` function in the url helper.  At first glance, however, it appears that you cannot use the `anchor()` function to link to external urls and thus have to write the anchor html element yourself.

**This is not true**. Instead of using the bare html element, I decided that I would extend the helper to include the functionality I wanted.  When I looked into the helper, though, it appears that the CodeIgniter team had already done it!  However, as of CI 1.7.2, the functionality is **undocumented**. Luckily, it is easy to do.

    <?php echo anchor("http://www.codeigniter.com", "CodeIgniter"); ?>

<!--more-->

Instead of specifying a controller and function to link to, supply the anchor() function with a full qualified domain address (http://...).  The helper will write the correct link and you won't have to do it by hand.

Do you have any other undocumented tips or tricks with the standard CodeIgniter library?  I would love to hear what they are!

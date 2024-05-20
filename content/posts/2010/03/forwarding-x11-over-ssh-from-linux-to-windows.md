+++
title = 'Forwarding X11 over SSH from Linux to Windows'
date = 2010-03-19T15:00:00-05:00
draft = false
tags = ['linux', 'youtube', 'ssh']
+++
For those that are not familiar with Linux, X11 is a part of the graphics system, and that is really all you need to know for this short blog entry.  Below is a video that demonstrates what forwarding x11 over SSH will actually allow you to do: in the video, I run a remote session of Kate (a Linux text editor) and Gwenview (a Linux image viewer) from my Linux machine to my Windows machine.  It's similar to remote desktop, but instead of the whole desktop, you can use certain applications.  You'll see what I mean in the video if you're not following already.

The first step is to configure SSH client and server on your Linux machine.  Instead of repeating what's been written a thousand times before, I'll send you to the [Ubuntu SSH Server instruction guide](https://help.ubuntu.com/9.04/serverguide/C/openssh-server.html).  If you use a different distribution, you can look up it for yourself if you have to, but as far as SSH goes it is pretty much the same wherever you go.

<!--more-->

Once you've gotten SSH server installed and running, double check your configuration file and make sure you have GUI forwarding turned on.  You can do this by finding the line that says "X11Forwarding yes" and uncommenting it if it is commented out (has a # in front of the line).  If the line is not in the file, add it, save the file, and restart your SSH server.

The video below will give a small tutorial about running the forwarded applications on your Windows machine.  You'll need [Putty](http://www.chiark.greenend.org.uk/~sgtatham/putty/) and [Xming](http://www.straightrunning.com/XmingNotes/) to do it.

This video is an oldy but a goody; I did it for a class in my final semester of college.  Expect more (and better) screencasts like this in the future.  Without further ado, away we go...

{{< youtube NNuXpk10zXE >}}

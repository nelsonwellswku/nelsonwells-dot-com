+++
title = 'Sometimes You Should Just Go Look'
date = 2014-07-14T15:00:00-05:00
draft = false
tags = ['programming', 'distraction']
+++
Fair warning: I shamelessly stole this story from a colleague.  Let's call him Jimmy. He doesn't have a web presence though, and I think it's worth sharing.

Jimmy was working on a web application. This particular web application had a feature that only activated on a double-click.  Don't be too harsh on Jimmy; he didn't have a choice in the matter. Sometimes, you have to do what you have to do.

<!--more-->

A user, let's call her Sandra, was having trouble with this particular feature and was having trouble communicating why. A ticket would come into Jimmy's work queue with a description akin to "it don't work [sic]" and nothing else. Jimmy tried his best to recreate the problem with no success. Every browser that the company officially supported seemed to work correctly with the Javascript responsible for identifying a double-click.  He sent the ticket back to Sandra looking for more information.

Somehow, the ticket bounced back and forth between the two a few times with no suitable reproduction steps, and Jimmy was at his wit's end. After several code re-writes and testing across both supported and unsupported browsers, Jimmy gave up and asked Sandra if he could see the problem in action.  Once he found Sandra's desk, he asked for a demonstration.

Sandra brought up the UI and found the feature in question.  Jimmy was bristling with excitement, at least until he saw the problem.  Sandra hovered her cursor over the button in the UI and clicked with fierce tenacity. A moment later, and no sooner, she clicked again, with equal tenacity.  A wry grin must have crossed Jimmy's face as he explained the difference to Sandra between a double-click and two clicks in succession.

If you learn nothing from my posts, learn these two lessons:

1. Never put a double-click event in a web app
2. Walk to the user's desk to identify mysterious problems that only they seem to have

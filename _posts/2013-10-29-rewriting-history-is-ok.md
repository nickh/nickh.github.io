---
layout: post
permalink: rewriting-history-is-ok.html
title: How I Stopped Worrying and Learned to Rewrite History
category: work
tags: github
---

I've been a long-time fan of git because of the workflows that it makes not just possible
but easy. I recently dealt with an app issue that highlights why these features make me
smile.

# Part I: Down The Rabbit Hole

When I'm digging into problems, I like to stay focused on the code as much as possible.
I expect to follow leads that don't work out, rewind, and try again in different directions. It can
be a confusing process to keep in your head and when you lose track of where you are and how you got
there it can be quite time-consuming to get back. Keeping the interruptions to a minimum is a huge
help, and that includes tools that aren't part of the debugging process - like source control.

The reported issue was that certain requests were failing after a long timeout. I wasn't
sure what the problem was or how I'd be fixing it, but I suspected that it would take some trial
and error, so I started by making myself a place to play around:

````sh
nictocat:app nickh$ git fetch
remote: Counting objects: 1187, done.
remote: Compressing objects: 100% (394/394), done.
remote: Total 1083 (delta 769), reused 987 (delta 676)
Receiving objects: 100% (1083/1083), 3.89 MiB | 1.90 MiB/s, done.
Resolving deltas: 100% (769/769), completed with 59 local objects.
From https://github.com/github/slumlord
   bd736bf..f7d42be  master     -> origin/master

nictocat:app nickh$ git rebase origin/master
First, rewinding head to replay your work on top of it...
Fast-forwarded master to origin/master.

nictocat:app nickh$ git checkout -b faster_updates
Switched to a new branch 'faster_updates'
````

At this point, I felt free to go crazy debugging this thing - knowing that if I needed to put
this on hold it would be easy to save what I was up to and bounce back to `master` to work with
the latest deployed version of the app.

I started by adding some profiling calls and found a few areas where caching started to improve
performance. I wasn't close to finding a root cause yet and wanted to stay focused on digging
further into the problem, but I also didn't want to lose my work; so I made a small checkpoint
commit to make it easy to get back to this point:

````sh
nictocat:app nickh$ git commit -am wip
[faster_updates e4b5751] wip
 2 files changed, 1 insertion(+), 2 deletions(-)
````

I kept digging and shaved off a couple more seconds by finding a few more things to cache, and
each time made another 'wip' commit.

Eventually I found the problem - one line of code that was filtering a Ruby Hash using `Array#include?`.
Changing the `Array` to a `Set` cut a 7-second operation down to 0.01 seconds, and since the method was
called twice we were talking about at least 14 seconds for each request. Another 'wip' commit and I
felt like I had a solid handle on the problem and could start thinking about cleanup.

Now that I knew where the big performance hit had been, I was curious whether any or all of the
caching changes I'd made were still necessary.  I used `git log -p` to get an overview of the
changes I'd made along the way, and one at a time undid each change to see how it affected
performance. The steps looked something like this:

* Edit the code to undo a change
* Time a request
* Use `git stash save` to temporarily undo the undo
* Time a request
* Use `git stash pop` to temporarily redo the undo

This let me easily see the performance impact of each of the smaller caching change on top of
the `Set` fix, and it turned out that all but one of the caching changes were unnecessary.
After timing each change I used either `git commit -am wip` to save the code without the
unnecessary change, or `git checkout <file> ...` to keep the change.

# Part II: Make It Public

Now it was time to start thinking about sharing the changes. `git log -p` looked pretty messy,
with a few profiling calls and comments in the code, and completely useless commit messages.
When I push a change, I know that it's for future me and my coworkers to figure out what changed
and why - and "wip" is just not helpful.

I got rid of the profiling calls and comments, made another checkpoint with `git commit -am cleanup`,
and then rebased. Rebasing rewrites history, which is sometimes considered harmful or bad form;
but at this point I hadn't pushed anything - and the rewrite was going to leave the history in a
much more polite state.

The git rebase command gives you a lot of control over how you want the
history to change - in this case I wanted to put all the changes in a single commit
with a useful commit message.

The commit log started out like this:

````sh
nictocat:app nickh$ git log
commit c85c4d5e9b4f1bb400cd73f5ea4283dda75bd777
Author: Nick Hengeveld <nickh@github.com>
Date:   Mon Nov 4 11:07:49 2013 -0800

    cleanup

commit 3cf9f53a2477bea4ea3bb15992c492d193e21c32
Author: Nick Hengeveld <nickh@github.com>
Date:   Mon Nov 4 10:54:37 2013 -0800

    wip

commit 753ff63643b002e584f6f5113bc9a669f347f0e1
Author: Nick Hengeveld <nickh@github.com>
Date:   Mon Nov 4 10:39:30 2013 -0800

    found it

commit be73397ed564ad7fc649fa816f2dd965cfa8a0a0
Author: Nick Hengeveld <nickh@github.com>
Date:   Mon Nov 4 10:08:11 2013 -0800

    wip

commit 72d3f94e26cd251cc70da12e222a3f1536a9e187
Author: Nick Hengeveld <nickh@github.com>
Date:   Mon Nov 4 09:47:04 2013 -0800

    wip

commit 620d0f3b2b6ea33667fa8609316769de4fe9497a
Author: Nick Hengeveld <nickh@github.com>
Date:   Mon Nov 4 11:06:52 2013 -0800

    wip
````

I ran `git rebase -i origin/master` to start an interactive rebase and told it how to rewrite my
commits:

````
  1 reword ea7ccf0 wip
  2 fixup 620d0f3 wip
  3 fixup 72d3f94 wip
  4 fixup be73397 wip
  5 fixup 753ff63 found it
  6 fixup 3cf9f53 wip
  7 fixup c85c4d5 cleanup
  8
  9 # Rebase c0a3667..c85c4d5 onto c0a3667
 10 #
 11 # Commands:
 12 #  p, pick = use commit
 13 #  r, reword = use commit, but edit the commit message
 14 #  e, edit = use commit, but stop for amending
 15 #  s, squash = use commit, but meld into previous commit
 16 #  f, fixup = like "squash", but discard this commit's log message
 17 #  x, exec = run command (the rest of the line) using shell
````

I saved the rebase instructions and typed in a new commit message, and now my log looked like this:

````sh
nictocat:app nickh$ git log
commit 87246e0f353ae32cddc580b6a806c9c36efc9a41
Author: Nick Hengeveld <nickh@github.com>
Date:   Mon Nov 4 11:07:49 2013 -0800

    Fixed update report request timeouts

    The method that filters missing commits from the map was taking
    a very long time because the Array lookups were slow; changing to
    a Set reduces request time by several orders of magnitude.

    In addition, caching the relative path history for a specific
    directory prefix saves several seconds per request.
````

Now I felt confident that someone viewing this code weeks or months from now would have a good idea
of what changed and why. For example, this is the only place in the application where a `Set` is used,
and someone refactoring a year from now may consider using an Array for consistency; a quick look with
`git blame` would inform them that it is different for a good reason.

At this point, I could have used `git push origin faster_updates` to share my changes with the rest of
them team, but experience has taught me to be paranoid so I ran `git log -p` to take a closer look at
what I was about to share. Sure enough, there were a couple of debug statements left in there. I 
removed them, ran `git commit -am fixup` and `git rebase -i HEAD^^` to "fixup" the commit I was about
to push. Another review of `git log -p` looked clean, so I pushed and opened a pull request.

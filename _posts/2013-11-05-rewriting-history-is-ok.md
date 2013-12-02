---
layout: post
permalink: rewriting-history-is-ok.html
title: How I Learned To Stop Worrying And Love Rewriting History
category: work
tags: github
---

I've been a long-time fan of Git because of the workflows that it makes both possible
**and** easy. I recently dealt with an application issue that highlights why these features make me
smile.

### Part I: Down The Rabbit Hole

When I'm digging into problems, I like to stay focused on the code as much as possible. I expect to
follow leads that don't work out and then rewind and try again in different directions. It can be a
confusing process to keep in your head. When you lose track of where you are and how you got there,
it can be quite time-consuming to get back to where you were. Keeping the interruptions to a minimum
is a huge help, and that includes using tools that aren't part of the debugging process, such as a
version control system (VCS).
 
The reported issue was that certain requests were failing after a long timeout. I wasn't sure what
the problem was or how I'd fix it, but I suspected that it would take some trial and error, so I
started by making myself a place to play around:

{% highlight sh %}
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
{% endhighlight %}

At this point, I felt free to go crazy debugging this thing - knowing that if I needed to put
this on hold it would be easy to save my work in progress and switch back to the master branch
to work with the latest deployed version of the app.

I did some profiling and found a few areas where additional caching improved performance a bit.
I wasn't close to finding a root cause yet and wanted to stay focused on digging
further into the problem. I also didn't want to lose my work, so I made a small checkpoint
commit to make it easy to get back to this state of the code:

{% highlight sh %}
nictocat:app nickh$ git commit -am wip
[faster_updates e4b5751] wip
 2 files changed, 1 insertion(+), 2 deletions(-)
{% endhighlight %}

I kept digging and shaved off a couple more seconds by finding a few more things to cache, and
each time made another 'wip' (work in progress) commit.

Eventually I found the problem - one line of code that was filtering a Ruby `Hash` using `Array#include?`.
Changing the `Array` to a `Set` cut a 7-second operation down to 0.01 seconds. Since the method was
called twice, we were talking about at least 14 seconds for each request. Another 'wip' commit and I
felt like I had a solid handle on the problem and could start thinking about cleaning up the change history.

Now that I knew where the code that caused the big performance hit lived, I was curious whether any or all of the
caching changes were still necessary. I used `git log -p` to get an overview of the
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

### Part II: Make It Public

Now it was time to start thinking about sharing the changes. `git log -p` looked pretty messy,
with a few profiling calls and comments in the code, and completely useless commit messages.
When I push a change, I know that part of the value is to remind my coworkers and I in the future of what changed
and why it changed. Commit messages with "wip" are not helpful.

There are a couple of ways that I frequently clean up history with git:

* Use `git reset --soft` to leave the code in its current state but
  erase some commit history. After running this command you have several options, such as making one
  new commit or incrementally adding related changes into discrete
  commits.
* Use `git rebase -i` to play back a set of commits with modifications,
  such as grouping them together, reordering them, removing them, or
  changing commit messages.

In either case I'm comfortable that I won't lose anything. Git maintains
a "reflog" that I can always use to get back to a previous state. If
I start making changes and change my mind or mess things up, it's no big
deal. In this case, I opted for a rebase because it's a very quick and
easy way to squash several changes together into a single commit.

I got rid of the profiling calls and comments, made another checkpoint with `git commit -am cleanup`,
and then rebased. Rebasing rewrites history, which is sometimes considered harmful or bad form.
At this point I hadn't pushed anything and the rewrite was going to leave the history in a
much more polite state, so the usage was in fact, good form.

The git rebase command gives you a lot of control over how you want the
history to change - in this case I wanted to put the following changes into a single commit
with a more useful commit message.

The commit log started out like this:

{% highlight sh %}
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
{% endhighlight %}

I ran `git rebase -i origin/master` to start an interactive rebase and told it how to rewrite my
commits:

{% highlight sh linenos %}
reword ea7ccf0 wip
fixup 620d0f3 wip
fixup 72d3f94 wip
fixup be73397 wip
fixup 753ff63 found it
fixup 3cf9f53 wip
fixup c85c4d5 cleanup

# Rebase c0a3667..c85c4d5 onto c0a3667
#
# Commands:
#  p, pick = use commit
#  r, reword = use commit, but edit the commit message
#  e, edit = use commit, but stop for amending
#  s, squash = use commit, but meld into previous commit
#  f, fixup = like "squash", but discard this commit's log message
#  x, exec = run command (the rest of the line) using shell
{% endhighlight %}

I saved the rebase instructions and typed in a new commit message, and now my log looked like this:

{% highlight sh %}
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
{% endhighlight %}

Now I felt confident that someone viewing this code weeks or even months from now would have a good idea
of what changed and why. For example, this is the only place in the entire application where a `Set` is used,
and someone refactoring a year from now might consider using an `Array` for consistency. With this more articulate history, a quick review with
`git blame` would inform them that a `Set` was chosen and used for a good reason.

At this point, I could have used `git push origin faster_updates` to share my changes with the rest of
the team, but experience has taught me to be paranoid. I ran `git log -p` to take a closer look at
what I was about to share. Sure enough, there were a couple of debug statements left in there. I
removed them, ran `git commit -am fixup` and `git rebase -i HEAD^^` to "fixup" the commit I was about
to push. Another review of `git log -p` looked clean, so I pushed and opened a pull request.

The polished branch now contained all the benefit of the code changes, but additionally was made
even more valuable with the polished history that told a story of "why" this change was made.
This makes the branch a better learning tool for future team members, less at risk of being mistakenly
reverted to an `Array`, and clearer to the reader during any potential seasonable change auditing efforts.
I've always felt that rewriting history with Git is a great investment, made easy and effective with
the tool's support of this workflow.

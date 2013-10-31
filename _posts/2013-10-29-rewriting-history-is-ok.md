---
layout: post
permalink: rewriting-history-is-ok.html
title: How I Stopped Worrying and Learned to Rewrite History
category: work
tags: github
---

I've been a fan of git for a long time because of the workflows that it makes not just possible
but easy. I recently dealt with an app issue that highlights why these features make me
smile.

The reported issue was that certain requests were failing after a 30-60 second timeout. I wasn't
sure what the problem was or how I'd be fixing it, but I suspected that it would take some trial
and error, so I made myself a place to play around:

````
git fetch
git checkout -b faster_updates origin/master
````

At this point, I felt free to go crazy debugging this thing - knowing that if I needed to put
this on hold it would be easy to save what I was up to and bounce back to `master` to work with
the latest deployed version of the app.

I started by adding some profiling calls and found a few areas where caching values started to
improve performance. I wasn't sure yet whether I'd hit the root cause but I was starting to make
progress, so I made a checkpoint so it would be easy to get back to this point:

````
git commit -am wip
````

I was nowhere near ready to share this with anyone. I wasn't sure I'd found everything, or even
the right thing, so it seemed silly to try to describe what I'd changed or why.

I shaved off a couple more seconds by finding a few more things to cache, and each time made
another 'wip' commit.

Eventually I found the problem - one line of code that was filtering a Ruby Hash using `Array#include?`.
Changing the Array to a Set saved 7 seconds, and the method was called twice so we were talking about
at least 14 seconds for each request. Another 'wip' commit and I felt like I had a good handle on
the problem and could start thinking about cleanup.

Now that I knew where the big performance hit had been, I was curious whether any or all of the
caching changes I'd made were still necessary.  I used `git log -p` to get an overview of the
changes I'd made along the way, and one at a time undid each change to see how it affected
performance:

````
# time a request without the caching change
git stash save
# time it again with the caching change
git stash pop
# time it again without the caching change
````

It turned out that all but one of the caching changes were unnecessary after fixing the filter
problem. After looking at each change I used either `git commit -am wip` to undo the changes
I didn't need or `git checkout <file> ...` to keep the change.

Now it was time to start thinking about sharing the changes. `git log -p` looked pretty messy,
with a few profiling calls and comments in the code, and completely useless commit messages.
When I push a change, I know that it's for future me and my coworkers to figure out what changed
and why - and "wip" is just not helpful.

I cleaned up the profiling calls and comments, did another `git commit -am wip`, and then rebased.
Rebasing rewrites history, which is sometimes considered harmful or bad form, but at this point
I hadn't pushed anything - and the rewrite was going to leave the history in a much more polite
state after I did. The git rebase command gives you a lot of control over how you want the
history to change - in this case I wanted to put all the changes in a single commit
and update the commit message:

````
git log
# lots of wip commits

# rebase overview, fixup and amend

git log
# one beautiful commit
````

At this point, I could have used `git push origin faster_updates`

There were still a few profile calls and comments I'd added during

I started using git to manage source code in 2006. In 2009, I interviewed at a Subversion shop
and one of the engineers asked why I preferred git to Subversion; I answered that git separates
the actions of making checkpoints as your code changes and of sharing those changes.
...

Recently fixed some perf issues where this workflow came in handy.
- open issue, certain types of requests timing out frequently
* part one: making checkpoints, easy to rewind/toggle
- started a branch
- didn't know where the problem was/what the fix was
- started profiling, found some areas where additional caching improved performance
- git commit -am wip
- found more, repeat
- found the big one, fixed it
- git commit -am wip
* part two: looks good, now think about sharing it
- the commit is for future me and future coworkers to know what changed here and why
- lots of 'wip' comments are the opposite of helpful, and are littered with debug output
- add commits that undo the debugging
- smoosh the commits together
- realize that early optimizations may not be necessary
- edit the code to undo some changes
- test, git stash save, test again, git stash pop
- remove bits that aren't needed
- git log -p, figure out what changes i'm about to push
- realize there are things i meant to delete, things that don't make sense etc.
- edit, git commit -am fup, git rebase -i HEAD^^, fixup
- repeat
- commit diff looks good, :thumbsup:
- commit message is still 'wip'
- git commit --amend, leave a useful message for posterity
- git push

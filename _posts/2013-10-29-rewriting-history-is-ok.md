# Rewriting History Is OK

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

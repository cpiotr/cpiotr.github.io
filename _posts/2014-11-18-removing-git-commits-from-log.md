---
ID: 230
post_title: Removing git commits from log
author: Piotr Ciruk
post_excerpt: ""
layout: post
permalink: >
  http://ciruk.pl/2014/11/removing-git-commits-from-log/
published: true
post_date: 2014-11-18 21:44:34
---
A perfect balance when everything is green and refactored. This sweet spot that happens from time to time. Enjoyable harmony that urges you to <code>git commit</code> your code.
Heck, you can even <code>git push</code> it, shut down the machine and head home.
Back at home you slowly realize. You committed one file too many, altered few things that were supposed to remain untouched. A vision of a relaxing evening fades away.
Unless of course you can go back and remove entries disgraceful log entries.

Git makes it as easy as pie.

To remove one commit locally use:
[sourcecode lang="bash"]
git reset HEAD^
[/sourcecode]

To vanish last 14 commits slightly modify the previous one:
[sourcecode lang="bash"]
git reset HEAD~14
[/sourcecode]

And the following command forces your changes to remote repository:
[sourcecode lang="bash"]
git push origin +HEAD
[/sourcecode]
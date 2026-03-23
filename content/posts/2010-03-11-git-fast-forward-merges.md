---
title: Git Fast Forward Merges
date: 2010-03-11
comments: true
slug: "git-fast-forward-merges"
tags:
  - git
---

Moving our current projects codebase from Subversion to Git was a nice move. This plus the adoption of the feature-centric way of developing (BDD + Scrum + Kanban) and our repositories are now cleared of any form of waste (useless LOC written "just in case"). Now, we must adapt the usage we have of Git and one that we just initiated is the non fast forward merge.

Moving our current projects codebase from Subversion to Git was a nice move. This plus the adoption of the feature-centric way of developing (BDD + Scrum + Kanban) and our repositories are now cleared of any form of waste (useless LOC written "just in case").


Now, we must adapt the usage we have of Git and one that we just initiated is the non fast forward merge.


Cheap branches with Git are great and it's a pleasure to work with them for both new implementations and quick experimentations. Behavior-Driven Development and agile methodologies don't have to prove they work anymore: it's just our only way of working now (which is IMHO good, not because I am responsible for its adoption, but because it actually _works_).


Being a small company we don't have so many branches, except master (for the continuously integrated trunk) and few tags (for deployed or "soon-deployed-yet-to-be-optimized" versions). But it's quite simple to dive into the workflow, fork any branch and merge it back when done ("done" meaning: tests are still turning green).
For a longer feature, if breaking the implementation in smaller tasks wasn't possible, it's easy to merge often the master to the feature's branch to avoid the big bang integration resulting from a branch too long away from its parent.


In one of our projects, we wanted to experiment two different ways of coding the same feature. Easy to do, easy to figure out what branch to keep and merge to the master branch. Having low activity on our codebase, Git merges are fast-forward merges.


From the Git documentation:


#### Fast-forward merges
```
[...] Normally, a merge results in a merge commit, with two parents, one pointing at each of the two lines of development that were merged.
```

However, if the current branch is a descendant of the other—so every commit present in the one is already contained in the other—then git just performs a "fast-forward"; the head of the current branch is moved forward to point at the head of the merged-in branch, without any new commits being created.


So when we merged back, Git just doesn't care about the branch we created and pretend that it never existed and so we can't see anywhere in the log we entered "git merge". At that time, *working on master would have had the exact same effect*. If we ask to ourselves "did we create a branch for that?" a couple of weeks after the development, we can only assume we did (as that's our only - allowed - way to work).


What's happening now we would like to revert that feature? We must read all the history searching for the very first change involved in that feature (and as things are so perfectly aweful the commits are not only focusing on the feature we implemented, so we must alread read all the code changed) and revert changes carefully each one at a time...


So now, not only we do branches but also we do non fast forward merges to ensure we see the merge in the history.


```
git merge --no-ff
```


As a result, it's now easy to spot what is the involved changeset, and also we can just revert the *entire branch* easily.

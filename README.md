git-tbdiff: topic branch interdiff
==================================

Installation:

    cp git-tbdiff.py /usr/local/bin/git-tbdiff
    # or anywhere else in $PATH, or in $(git --exec-path)

If your system does not yet have a /usr/bin/python2 symlink (older
systems would only have /usr/bin/python), you will need to edit the
`#!` line.

Usage:

    git tbdiff A..B C..D

to compare the topic branch represented by the range A..B with that in
the range C..D.

We do not have any convenient tools so far for seeing the difference
between versions of a topic branch.  Some approaches seen in the wild
include:

* use git-cherry as a first-order comparison

* rebase the old version on the new version to a) have the patch-id
  logic drop equivalent patches and b) [usually] get a conflict when
  the patches themselves differ on a change

* apply on the same base

* run interdiffs across the series

* run an interdiff of the "squashed diff" (base to branch)

We propose a somewhat generalized approach based on interdiffs.  The
goal would be to find an explanation of the new series in terms of the
old one.  However, the order might be different, some commits could
have been added and removed, and some commits could have been tweaked.

The general idea is this:

Suppose the old version has commits 1--2 and the new one has commits
A--C.  Assume that A is a cherry-pick of 2, and C is a cherry-pick of
1 but with a small modification (say, a fixed typo).  Visualize the
commits as a bipartite graph:

    1            A

    2            B

                 C

We are looking for a "best" explanation of the new series in terms of
the old one.  We can represent an "explanation" as an edge in the
graph:


    1            A
               /
    2 --------'  B

                 C

The 0 represents the edge weight; the explanation is "free" because
there was no change.  Similarly C can be explained using 1, but it has
some cost c>0 because of the modification:


    1 ----.      A
          |    /
    2 ----+---'  B
          |
          `----- C
          c>0

Clearly what we are looking for is some sort of a minimum cost
bipartite matching; 1 is matched to C at some cost, etc.  The
underlying graph is in fact a complete bipartite graph; the cost we
associate with every edge is the size of the interdiff between the two
commits in question.  To also explain new commits, we introduce dummy
commits on both sides:

    1 ----.      A
          |    /
    2 ----+---'  B
          |
    o     `----- C
          c>0
    o            o

    o            o

The cost of an edge o--C is the size of C's diff.  The cost of an edge
o--o is free.

This definition allows us to find a "good" topic interdiff among
topics with n and m commits in the time needed to compute n+m commit
diffs and then n*m interdiffs, plus the time needed to compute the
matching.  For example, in this Python version we use the hungarian[1]
package, where the underlying algorithm runs in O(n^4)[2].   The
matching found in this case will be like

    1 ----.      A
          |    /
    2 ----+---'  B
       .--+-----'
    o -'  `----- C
          c>0
    o ---------- o

    o ---------- o

Then we reconstruct a "pretty" (well, not quite) output that
represents the topic diff.



[1]  https://pypi.python.org/pypi/hungarian

[2]  http://en.wikipedia.org/wiki/Hungarian_algorithm
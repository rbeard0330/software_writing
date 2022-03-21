---
layout: post
date: 2022-03-20
title: "Cracking Cracking the Coding Interview, Part 2"
---

Consider the following problem from *Cracking the Coding Interview* by Gayle Laakman McDowell:

> **4.10 Check Subtree:** T1 and T2 are very large binary trees, with T1 much bigger than T2.  Create an algorithm to determine if T2 is a subtree of T1.
> 
> A tree T2 is a subtree of T1 if there exists a node n in T1 such that the subtree of n is identical to T2.  That is, if you cut off the tree at node n, the two trees would be identical.

The book offers two solutions:

1. The first approach (which McDowell calls "the simple approach") is to represent both T1 and T2 as strings, then see if T2 is a subtree of T1.  The representation that turns out to work is a pre-order traversal, with explicit representation of the empty nodes. Using a naive searching algorithm, this solution has time complexity O(mn), where m is the size of T2 and n is the size of T1.  Faster algorithms, however, could improve this to O(n).  This approach uses space of O(n + m), which is bounded above by O(n) since T1 is larger than T2.
2. The second approach (which I'll call Root-Match Search) is to do a preorder traversal of T1, invoking a matching subroutine whenever the value at the current node equals the value of the root of T2.  The worst-case runtime for this algorithm is also O(mn), though the book argues that you can usefully restate this as O(n + km), where k is the number of times the root element in T2 appears in T1.  In other words, you need O(n) steps to traverse T1, and you incur a cost of up to m every time you encounter a false-positive match on the root value of T2.  Space complexity is O(log n) for the stack used to manage the traversal, if the tree is balanced, but O(n) if it is not.

My initial approach on reading the problem was to do a variation of approach 1, without the intermediate step of constructing a string representation.  In other words, start independent pre-order traversals of both trees until you get to the left-most nodes in T1 and T2 (call this node in T1 `L`).  Then proceed with these traversals in tandem.  If every step of the traversal of T2 is equivalent to the traversal in T1, then you have a match.  Otherwise, reset the traversal of T2 to the left-most child, and move the traversal of T1 back to the first node after `L` that has no left child.

Despite the fact that McDowell views the string search approach as the simpler alternative and that it was the first one to occur to me, I think the Root-Match Search is actually a simpler concept.  What could be more obvious than to just traverse the tree you're searching and do a brute force comparison every time the root matches?  Besides, trying to improve the first algorithm will entail a foray into the deep wilderness of search theory and finite state machines and who knows what else.  So let's push on Root-Match Search and see what we can come up with.

## The weakness of Root-Match-Search

The first thing to note is that the distinction between Root-Match Search and fully naive brute-force search where you invoke the subtree-matching routine on each node is actually just a matter of presentation.  To see this, consider what happens when you run the naive algorithm.  Say that the root value of T2 is `0`.  When wise Root-Match Search encounters a node `n` with a value of `1`, it compares `1` to `0`, realizes that they're different, and cleverly moves on to the next node in the traversal.  When large-adult-algorithm-son naive brute-force search encounters the same node, it just blindly invokes its subtree matching algorithm, like an idiot.  As a result, the subtree matching algorithm...compares the actual node value of `1` with the expected value of `0` and immediately terminates.  Huh. 

Let's consider the situations where RMS struggles.  The worst case situation would be one where all the values in the tree are identical, and we're just matching the structure of the two trees.  The reason this is so bad for RMS is that every node matches the root, and we have to resort a full subtree-matching process on each node we see.  What's going on here is that the tree contains both node values and structure, and we're only considering node values (more precisely, the root node value) when we decide which nodes in T1 could be the root of a T2 subtree.

We need a better screening process.  While the value stored at a potential root is very easy to obtain, it is otherwise a *terrible* way to screen:
1. The value at the root of the subtree encodes nothing whatsoever about the rest of the values in the subtree.
2. The value at the root of the subtree encodes nothing whatsoever about the structure of the subtree.
3. The full-match attempts that are screened out by root value checks are precisely the ones that don't matter to our runtime, because they terminate immediately.

Items 1 and 2 are crucial for a nonobvious reason.  If we start a full-match attempt at `n` because `n`'s value matched the root, we learn almost nothing if that full-match attempt fails.  In particular, **we still need to consider `n`'s descendants.**  This fact is what dooms us to an O(mn) runtime--we can do up to `m` work in a full-match attempt at a particular node, and all it tells us is that that specific node is not the root we're looking for.  We can end up looking at the same node over and over against as an element of many different subtrees.  We need to eliminate that possibility to make progress.     

## Screening on whole-subtree properties

We need a richer metric to tell us which nodes to investigate as potential T2 roots.  In particular, we need statistics about the complete subtree rooted at each node `n` *before* we decide to do a full investigation of that node.

...

Wait, how can we investigate a subtree before we investigate it? Isn't that a contradiction?  Well, yes and no.  The specific kind of investigation that's too time-consuming to do a lot of is trying to match a subtree against T2.  The reason that's so costly is that the information we get from the procedure relates solely to the potential root.  But notice that most tree properties aren't like that, which is what makes trees such interesting structures.  Most tree properties (at least, most interesting ones) have a recursive structure.  Consider how you would compute the sum of the values in a tree.  The most natural way is something like this:

```python
def tree_sum(node):
    if node is None:
        return 0
    else:
        return node.value + tree_sum(node.left) + tree_sum(node.right)     
```

To compute the sum of a node, we first recursively compute the sum of its children.  Or to put it another way, the process of computing the sum of the root naturally computes the sum of every single subtree, all in one process.  *This* is what our screening process should look like.  We need to screen the whole tree in one pass, then drill down on a limited number of candidate nodes for a full-match attempt.

So what tree property should we use?  Using subtree-sum isn't a ridiculous idea, but it's not ideal.  To see why, imagine that the sum of the nodes in T2 is 100. Imagine further that we conclude that a subtree in T1 rooted at `n` also has a sum of 100.  The problem is that if we investigate `n` and it's not a match, we actually can't reject the entire subtree rooted at `n`.  There could be a subtree within `n` that *also* has a sum of 100.  (Trivially, the values at `n` and its left child could both be zero.)

Instead, consider *tree height* as an alternative.  Imagine that, as a preprocessing step, we annotate each node in T1 with its height:

```python
def compute_and_label_heights(node):
    if node is None:
        return 0
    node.height = max(compute_and_label_heights(n)
                      for n in (node.left, node.right)) + 1
    return node.height
```
We'll also compute T2's height from the root.  Now we can traverse T1 like this:
```python
def find_matching_subtree(node, t2, t2_height):
    if node is None or node.height < t2_height:
        return False
    elif node.height == t2_height:
        return are_trees_equivalent(node, t2)
    else:
        return any(find_matching_subtree(n, t2, t2_height)
                   for n in (node.left, node.right))        
```

We have two base cases--if we encounter a node that has a height less than T2, we simply terminate immediately.  If we have an exact match on height, we can feel justified invoking our `are_trees_equivalent` subroutine to check for a match in O(m) time.  Finally, if the current node is too high, we just recurse and look for something shorter.

The critical observation here is that `are_trees_equivalent` is only invoked when the height is an exact match.  Because every non-root node in a subtree must have a lower height than the subtree root, this means that each call to `are_trees_equivalent` is on a node that is *not* a descendant of any other node that has had this routine called on it.  This means that `are_trees_equivalent` never looks at the same node twice, and the total amount of work done by all those calls is bounded above by O(n)![^1]

[^1]: Other structural properties could work here too.  For example, the total number of nodes ought to work just as well as height, since it preserves the critical invariant that the size of every subtree is strictly greater than the sizes of all of its sub-sub-trees.

Labelling the heights of the nodes is also an O(n) operation, and computing the height of T2 is O(m) < O(n).  Thus, the whole algorithm runs in simple O(n) time.

## A partially successful attempt to save space

An O(n) runtime is great, but the space complexity picture doesn't look as nice.  By storing the heights of every subtree, we use O(n) space, which feels like a lot for a search problem.  We ought to be able to economize here, because the only reason we care about the height is to trigger a full match attempt when we see the magic height.  Rather than storing the heights and finding the critical ones later, why not just try to do the match on the spot?

```python
def find_matching_subtree(t1, t2, t2_height):
    match_found = False

    def compute_height_and_trigger_searches(node):
        nonlocal match_found
        
        # Once a match has been found, bail as quickly as possible
        if node is None or match_found:
            return 0
        height = max(compute_height_and_trigger_searches(n)
                     for n in (node.left, node.right)) + 1
        if height == t2_height:
            match_found = match_found or are_trees_equivalent(node, t2) 
        return height

    compute_height_and_trigger_searches(t1)
    return match_found
```

The big cost here is a meaningful hit to code clarity--`compute_height_and_trigger_searches` just openly confesses to crimes against the single responsibility principle.  The appearance of the rarely seen `nonlocal` keyword is another red flag.[^2]  The basic problem is that the recursive calls need to return two totally different pieces of information.  In addition to returning the heights of the subtrees, we also need to communicate the results of our match attempts.  There's no perfectly elegant solution in Python[^3], but opening a side channel in the form of the `match_found` variable is a workable, if flawed, answer.

[^2]: For any virtuous souls who avoid writing code that needs to know about it, `nonlocal` is needed because we're modifying a variable that lives outside the local scope of the `compute_height_and_trigger_searches` function. The normal rule in Python is that a function can access such variables (including calling mutating methods), but can't reassign them.  The `nonlocal` keyword tells Python that we've decided to violate that rule. 

[^3]: In a language with algebraic data types, it would probably be better to return a sum type with two variants (`SearchResult` vs `SubtreeHeight`), since we never actually need to return both types of information from the same call.  We could emulate this in Python by return a Golang-y tuple consisting of a tag for the kind of result and the actual value if we chose, but that feels messier to me.

This rewrite reduces our space costs down to the call stack for the recursive `compute_height_and_trigger_searches` calls.  This is O(h), which is equivalent to O(log n) for a balanced tree, but could be as high as O(n) for a pathologically unbalanced example.  In theory, it ought to be possible to defeat the pathological case.  The fundamental problem with the pathological case is that the nodes don't branch and each recursive call spawns another call with a tree of almost exactly the same size, which creates a new entry in the call stack.  In our case, though, the call stack entry isn't really needed.  For the branching case, the call stack handles the bookkeeping needed to keep track of the different pieces of the calculation. For the non-branching case, no such bookkeeping is required, so the call stack entry is dead weight.  Concretely, if `node` has only one descendant `c`, we can rewrite this line of code:

```python
height = max(compute_height_and_trigger_searches(n)
             for n in (node.left, node.right)) + 1
```
as:
```python
height = max(compute_height_and_trigger_searches(n)
             for n in (c.left, c.right)) + 2
```
More generally, we can skip over all single-child nodes and only recurse when we see a descendant with two children of its own.  Since we're no longer checking the height at each step, we also need to add a check to make sure we don't skip over the magic height.

```python
    def calculate_height_and_trigger_searches(node):
        nonlocal match_found

        current = node
        nodes_skipped = 0
        while child_count(node) == 1:
            nodes_skipped += 1
            current = current.left or current.right
        height = max(calculate_height_and_trigger_searches(n)
                     for n in (current.left, current.right)) + nodes_skipped + 1
        
        if height - nodes_skipped <= t2_height <= height:
            current = node
            # Loop does not execute if nodes_skipped = 0
            for _ in range(t2_height - (height - nodes_skipped)):
                current = current.left or current.right
            match_found = match_found or are_trees_equivalent(current, t2)
            
        return height
```

As modified, we only recurse on a branch.  Sadly, this isn't quite enough to cure the pathology.  Imagine a tree where each node has one child that's a leaf and one that contains the rest of its descendants. This code branches at every node, which still takes up too much space.  A full fix probably requires converting the recursion to a fully imperative style and aggressively controlling the bookkeeping data that we save.

## Using smarter fingerprints

When we settled on subtree height as the structural property to use as our filter for subtree root candidates, we didn't put a lot of thought into it.  All we needed was a property that encoded enough information to ensure that our candidate subtrees were disjoint.  That invariant gives us our O(n) runtime bound, but it still leaves the possibility that we run a *lot* of subtree matching attempts.  A tree of height h has up to 2<sup>h - k</sup> subtrees of height k, so we could be spending a lot of time searching.  We could reduce this cost by making our filter more clever.  Consider the following function:
```python
def fingerprint(node):
    if node is None:
        return None
    return hash((fingerprint(node.left), node.value, fingerprint(node.right)))
```
This code defines a node's fingerprint as the hash of a 3-tuple consisting of its own value and the fingerprints of its left and right children.  While we are computing tree heights, we could also compute the fingerprints for each node. This would allow us to only start a subtree match attempt when both the subtree height *and* the subtree fingerprint match. Absent hash collisions, our candidate screen should never get a false positive, so our first fingerprint match should be a winner.

Unfortunately, this bit of cleverness doesn't help us much on runtime.  The problem is that simply computing the heights for all the nodes in the tree takes O(n) time, which is a big enough time budget (asymptotically) to cover all the failed subtree match attempts we could ever have. Of course, it's still nice not to waste time on false-positive match candidates, but it may or may not balance out the extra work of computing the fingerprints.

It's not surprising that O(n) appears to be a hard lower limit on the runtime for a one-off search.  After all, we need to rule out at least n - m + 1 nodes in T1 before we can conclude that T2 isn't a subtree of T1.  If we don't, the last node we don't look at could be the root of an m-node subtree that's equivalent to T2.  And if we don't know anything about how T1 is organized (e.g., that it follows the search-tree or heap properties), there's just no way to rule out a node other than by examining it.  So O(n) is probably the best we can do.

The only situation where the fingerprint approach is really helpful is if we expect to run many searches against the same T1.  In that case, we can trade space for time and amortize the cost of computing the fingerprints over all the searches.  If we create a hash table index of all the nodes in T1, keyed by their height and fingerprints, we can search for a specific T2 by computing its height and fingerprint in O(m) time, then finding any potential match candidates with a constant-time has table lookup.  This search takes roughly O(m) time.[^4]  We also need O(n) space to store the index.

[^4]: Strictly speaking, the run time is probably better bounded by O(m + C(n, m)), where C(n, m) is a function that gives us the (presumably very small) number of expected hash collisions.
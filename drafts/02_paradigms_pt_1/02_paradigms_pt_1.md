# Shrinking algorithms

This essay is the first of several that will explore broad categories of algorithmic paradigms.  My goal is not to explain algorithms for particular problems, but rather to highlight broad principles and connections that might be helpful for others studying algorithms.  This essay will cover a category I think of as *shrinking algorithms* (which is not standard terminology). First we'll talk about what makes a shrinking algorithm a shrinking algorithm, then we'll talk about some paradigms where the concept appears.

## What is a shrinking algorithm?



## Divide and conquer (recursive flavor)

We start with a general template for solving a problem using divide-and-conquer:
```
def solve_with_divide_and_conquer(instance):
    if is_base_case(instance):
        return solve_base_case(instance)
    subproblems = divide_instance(instance)
    solved_subproblems = [solve_with_divide_and_conquer(subproblem) for subproblem in subproblems]
    return combine_solutions(solved_subproblems)
```

To use this template on a specific problem, we need to provide two substantive functions, and also handle identifying and solving the base case, which should be trivial.  The first substantive function we need takes a given instance and divides it into one or more smaller instances of the same problem.  The second takes a list of solved subproblems and combines them into a solution to the given problem.

This to-do list gives us a comprehensive list of the ways our attempted solution could fail:
1. We can't easily divide an instance into problems that are easier to solve; or 
2. Once we solve the subproblems, they don't give us a solution to the larger instance.  Alternatively, there's a way to use subproblems to find a solution, but you need so many subproblems that the D&C solution is too slow.

This list suggests dividing D&C algorithms into two categories, but I think it's more useful to split the second category into two parts, giving us three total buckets:
1. *Dividing Algorithms*.  For these algorithms, the real action is the first step, where we take the instance and divide it into smaller chunks for further processing.  Usually, this means that we aren't just shrinking the problem instance to make it easier to solve.  Instead, we're building special subproblems that will make our work easier in the combining step.
2. *Combining Algorithms*.  In this category, identifying the subproblems is easy (usually because we don't have any special requirements about how we divide the instance up).  The hard part is using the solved subproblems to get a complete solution.
3. *Subproblem Control Algorithms*.  In this category, it's not too hard to identify subproblems and use their solutions to solve the larger instance, but the simple approach is too slow.  Instead, the algorithm leverages some brilliant insight to solve the big problem using fewer or smaller subproblems, which speeds up the algorithm.

### Dividing Algorithms
This category is the rarest, at least in my experience.  When I started writing this essay, I actually planned to argue that the combining step was the only important part of the D&C paradigm.  But further consideration reveals that some algorithms do most or all their work in the dividing step, by picking ideal subproblems to solve.  The best example is quicksort.  Recall that in quicksort, the first step is picking a pivot element and reshuffling the input array so that all the elements in one subproblem are smaller than the pivot, and all the elements in the other are larger.  The payoff to doing this work up front is that when we get the solution to the subproblems back, the combine step is trivial.  (Indeed, for in-place quicksort, there is no combine step.  Defining the subproblems, then recursively defining their subproblems, etc., actually *is* sorting the array.)

Dividing algorithms can be harder to analyze, because the subproblems are defined dynamically, which makes it impossible to understand abstractly what the recursion tree for a particular instance size will look like.  In other words, calling merge sort on any list with 100 items will result in a call tree with a fixed size and shape.  By contrast (depending on how the pivot element is selected), quicksort will make a different number of recursive calls for different inputs, and could have a worse-case runtime greater than `n log n`.

#### Other examples
* [Linear selection](https://en.wikipedia.org/wiki/Quickselect)

### Combining Algorithms
This category is particularly important for technical interviews, since any D&C solution that is reasonably derivable in 45 minutes or an hour probably involves using the obvious subproblems and finding some way to combine those answers into a full solution.  There is also a broader sense in which the combining step really is a defining characteristic of a D&C algorithm. 


## Divide and conquer (limited flavor)
Recursive algorithms often feel like a bit of a swindle at first glance.  When you start reading through a description of merge sort, you're told to split the list in half, then sort each half.  If I already know how to sort a list, what are we even talking about?!  It's as if a recipe for chocolate cake included "bake a chocolate cake" as one of its steps.  Of course, we know that merge sort doesn't recurse infinitely.  Instead, it "sorts" a trivially sortable list in the base case of the recursion.  When the algorithm tells us to "sort" a subproblem, it is really telling us to subdivide it further and further till we get down to the base case.

It doesn't have to work this way.  Dividing a problem 


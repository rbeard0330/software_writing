---
layout: post
date: 2022-04-11
title: "Preserving Invariants in Rust with Drop"
---

Data structures can be divided into two categories, which I'll call *naive* and *aware* data structures (nonstandard terminology).  Naive data structures don't need to know anything about the data items they store.  As an example, consider the items in an array.  An array doesn't really care whether it's sorting integers, strings, other arrays, or complex objects.  All it needs to know is how much space a single item occupies, and then it can do its work by blindly moving appropriately sized groups of bytes around.  By contrast, an aware data structure needs to interact with the items it contains to function correctly.  Consider a binary search tree.  In order to determine where to store a particular item, the BST needs objects that can be compared with one another to produce a total ordering.

### Change can be hard

A difficult problem in the design of an aware data structure is whether and how to allow users of the structure to modify contained objects in place.  Hash tables, for example, are naive as to the values stored in the table, but aware as to the keys, which must be hashable in a stable way over their lifetimes.  To deal with this fact, Python's `dict`s require keys (but not values) to be "hashable."  In practice, only immutable Python primitives are hashable, and mutable user types that declare themselves hashable are responsible for ensuring that any mutations do not disrupt the hash-stability invariant.

There is not much reason to permit users to modify dictionary keys, so the limitations on `dict`s aren't a big deal.  For data structures that are aware of their *values*, though, it can be a bigger problem.  A standard implementation of [Dijkstra's algorithm](https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm) keeps currently reachable nodes in a min-heap, keyed by the shortest currently-known distance to that node.  At each step, the algorithm pops the smallest node from the heap and sets the distance to that node equal to its score.  Then, the heap is updated in place by checking each edge in the newly processed node and reducing the distance to any node that is reachable by a shorter path that runs through the newly processed node.

These modifications pose a serious problem for the heap!  To fulfill the Heap Oath it swore when it was heapified, the heap must always stand ready to yield its smallest value in constant time.  For that to be possible, the items in the heap need to be carefully ordered in a tree structure such that each node is the smallest value in its subtree.  A reduction in a value's key is very likely to break that invariant by making the modified value smaller than its parent.  If the heap wants to stay a heap, it must be able to respond immediately to any modifications, restore its invariants, and be ready to respond to the next method call.  How can this be arranged?

A simple, unsatisfactory solution is to make heap values immutable, but give clients the ability to completely delete a value from the heap.  The client can then modify the value and reinsert it.  While this works, it's awkward for the user and potentially inefficient.  After all, the change to the value might be insignificant enough that it doesn't even break the invariant.  In that case, doing the work to excise the item from the heap and insert the modified item is a pure waste.

Another solution is to restrict clients' access to stored data to specified operations.  Then the library author, knowing exactly what the client may have done, can ensure that any damage to invariants is cleaned up before the operation returns.  As an example, the textbook min-heap often exposes an operation `ReduceKey` that takes a given item and reduces its key (thereby moving the affected item closer to the top of the heap).  Since the data structure knows which object was affected and what the change was, it's relatively straightforward to fix up the heap invariant: all we need to do is invoke the standard heap `sift_up` procedure on the targeted item, and the invariant will be restored with a minimal amount of work.

While this solution is simple and hygenic, it's also somewhat limiting.  In particular, what happens if a heap item's key is not an independent property, but rather is derived from the other properties of the object?  For example, imagine that you're sending shipments of bulk commodities from a warehouse, and you want to dispatch the most valuable shipment first.  To manage this task, you might create a `Shipment` object with `commodity_value` and `shipment_size` attributes, and a custom property `shipment_value` that returns the product of those two attributes.  You'd like to store your `Shipments` in a max-heap, keyed by `shipment_value`, so that you can just pop a `Shipment` off the heap whenever a train is getting ready to leave.  But what happens now if rats eat half of a shipment of grain and you need to modify the `shipment_size` of the affected `Shipment` object, which is already in the heap?  A generic heap can't reasonably be expected to expose a method to modify your idiosyncratic objects.  You may have no better solution than to fall back to the unsatisfactory approach of removal, modification and reinsertion.

### A simple sample heap

I recently came across a rather elegant solution to this problem in Rust.  Consider the following `Heap` struct:

```rust
#[derive(Debug)]
pub struct Heap<T: HeapItem> {
    heap: Vec<T>,
    index_map: HashMap<Id, usize>,
}

pub trait HeapItem: Debug + Clone {
    fn key(&self) -> Key;
    fn id(&self) -> Id;
}

impl<T: Clone + Into<Key> + Into<Id> + Debug> HeapItem for T {
    fn key(&self) -> Key {
        self.clone().into()
    }
    fn id(&self) -> Id {
        self.clone().into()
    }
}
```

This heap stores any item that can produce a numeric key and id on request (or can be converted into such a type).  In addition to implementing the standard heap operations, we'd like to allow `HeapItem`s to be modified in place.  Exposing a `decrease_key` operation won't work, because we don't know how to modify the key of a generic `HeapItem`.[^1]  We could simply provide mutable access to clients: 

[^1]: Conceivably, we could extend the `HeapItem` trait to include a `set_key` operation that provides an interface for key changes. This could work, but it's not immediately obvious how it would help in our rat-infested `Shipment` example.  What we'd really need to handle that case is a full-blown interface for arbitrary modifications to a `HeapItem`, which is a ridiculous requirement just to allow an object to be stored in a container.

```rust
impl<T: HeapItem> Heap<T> {
    pub fn get_mut(&mut self, id: Id) -> Option<&mut T> {
        let index = *self.index_map.get(&id)?;
        &mut self.heap[index]
    }
}
```
To deal with broken invariants, we could ask the client to call a fix-up method after they're done modifying a value:
```rust
impl<T: HeapItem> Heap<T> {
    pub fn restore_invariants(&mut self, id: Id) {
        // check heap property at id and fix any violations
    }
}
```
This isn't entirely terrible, but we're pretty much giving up on having real confidence in the correctness of this heap.  It's simply not reasonable to expect that your users will always remember to call the fix-up method after they modify the data.  They *should*, but they just won't, at least not always.

### Smart pointers for smart people

Here's where the compiler can be drafted to help us out.  Our problem is that we want to let users modify their data directly, but we need to fix any problems they cause *after* they finish their modifications, but *before* any other user accesses the data structure.  Conveniently, that's (roughly--more on this later) when destructors are run!

Rather than returning a bare reference to the heap item, we can grant access via a smart pointer:

```rust
impl<T: Clone + Into<Key> + Into<Id> + Debug> HeapItem for T {
    fn get_mut(&mut self, id: Id) -> Option<HeapItemRefMut<T>> {
        let index = *self.index_map.get(&id)?;
        let original_key = self.heap[index].key();
        let original_id = self.heap[index].id();
        Some(HeapItemRefMut {
            view: self.get_mut_view_at(index),
            original_key,
            original_id,
        })
    }
}

struct HeapViewMut<'a, T: HeapItem> {
    index: usize,
    heap: &'a mut Vec<T>,
    index_map: &'a mut HashMap<Id, usize>,
}

struct HeapItemRefMut<'a, T: HeapItem> {
    view: HeapViewMut<'a, T>,
    original_key: Key,
    original_id: Id,
}

impl<'a, T: HeapItem> DerefMut for HeapItemRefMut<'a, T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        self.view.heap.get_mut(self.view.index).unwrap()
    }
}
```

Here, we actually use two layers of managed access--`HeapViewMut` is used internally for heap management and is where methods like `sift_up` live.  The client gets a wrapper struct that has no methods and just dereferences to the underlying heap item.  The advantage of using this rather complicated arrangement is that, when the smart pointer goes out of scope, it gets dropped and the compiler will insert a call to its destructor:

```rust
impl<'a, T: HeapItem> Drop for HeapItemRefMut<'a, T> {
    fn drop(&mut self) {
        println!("restoring invariants on reference drop");
        let new_id = self.view.heap[self.view.index].id();
        let new_key = self.view.heap[self.view.index].key();
        let (_, old_index) = self.view.index_map.remove_entry(&self.original_id).unwrap();
        debug_assert_eq!(old_index, self.view.index);
        self.view.index_map.insert(new_id, old_index);
        if self.original_key > new_key {
            self.view.sift_down();
        } else if self.original_key < new_key {
            self.view.sift_up();
        }
    }
}
```
We're completely indifferent to what the client did to the object.  All we need to do is check whether its key or id changed and make the appropriate changes if they did.  And most importantly, *the client can't forget to restore the invariants*!  They don't even need to worry about it.  That lets us have code like this:
```rust
#[derive(Clone, Debug)]
struct Job{
    priority: i64,
    id: i64,
    description: String
}

impl HeapItem for Job {
    fn key(&self) -> Key {
        self.priority
    }

    fn id(&self) -> Id {
        self.id
    }
}

#[test]
fn test_invariants_restored_automatically() {
    let mut job_queue = Heap::heapify(vec![
        Job {id: 1, priority: 100, description: "Very urgent!".to_string()},
        Job {id: 2, priority: 50, description: "Medium urgent!".to_string()},
        Job {id: 3, priority: 0, description: "Meh, whenever".to_string()}
    ]);
    println!("before modification");
    *job_queue.get_mut(3).unwrap() = Job {
        id: 3,
        priority: 200,
        description: "The boss wants this yesterday!".to_string()};
    println!("after modifications, before read");
    assert_eq!(&job_queue.pop().unwrap().description, "The boss wants this yesterday!");
    println!("after read");
}
```

When run, we see this lovely output:
```
$ cargo test
    Finished test [unoptimized + debuginfo] target(s) in 0.05s
     Running unittests (target\debug\deps\trapper_keeper-72f1867c5dd2762c.exe)
before modification
restoring invariants on reference drop
after modifications, before read
after read
```

Our fix-up code was automagically inserted right after the data was modified!  More specifically, because `HeapItemRefMut` holds a mutable, exclusive reference to the heap, Rust's aliasing rules will prevent any reads from the heap during the `HeapItemRefMut`'s lifetime.  This prevents any race conditions and ensures that the events always happen in the following sequence:
1. Client modifies heap item via the `HeapItemRefMut`.
2. The client finishes their work and the `HeapItemRefMut` is destroyed, which gives us a window to fix the heap invariants.
3. The next client interacts with the heap.

The aliasing rules ensure that no client can ever see the heap in an invalid state.

### The not so pretty parts
The explanation about the aliasing rules above is technically correct, but it oversimplifies things.  The problem is that destructors are *not* run at the end of an object's lifetime.  Instead, they run when the object goes out of scope. This is a blunter analysis than what the borrow checker uses to delineate reference lifetimes.  The following code compiles fine:

```rust
fn main() {
    let mut x = 1;
    let x_ref_mut = &mut x;
    *x_ref_mut += 1;
    // x_ref_mut's lifetime ends here
    let x_ref = &x;
    println!("{:?}", x_ref);
    // x, x_ref_mut, and x_ref are "dropped" here
}
```
It isn't a problem for the compiler that `x_ref_mut` hasn't been destroyed by the time `x_ref` is created and used. The borrow checker is smart enough to see that the last use of `x_ref_mut` occurs before a new reference to `x` is taken.  But that's not good enough for us!  We actually have a big problem if someone uses a reference after a heap item is modified, but before our clean-up destructor is run.  That's exactly what we're trying to prevent.

In the simple example above, we didn't have this problem, because the `HeapItemRefMut` was never bound to a variable, so it went out of scope at the end of the statement it was used in.  By contrast, if our clients save the pointer, we have a problem:

```rust
#[test]
fn invariants_not_restored_automatically() {
    let mut job_queue = Heap::heapify(vec![
        Job {id: 1, priority: 100, description: "Very urgent!".to_string()},
        Job {id: 2, priority: 50, description: "Medium urgent!".to_string()},
        Job {id: 3, priority: 0, description: "Meh, whenever".to_string()}
    ]);

    let mut handle = job_queue.get_mut(3).unwrap(); // Changed! 
    *handle = Job {
        id: 3,
        priority: 200,
        description: "The boss wants this yesterday!".to_string()};
    
    assert_eq!(&job_queue.pop().unwrap().description, "The boss wants this yesterday!");
    // handle is dropped here, which is too late
}
```

There's good news and bad news.  The bad news is that our code won't work in this situation.  As far as I know, there's no way to game the drop scope rules to force the compiler to drop `handle` at the end of its lifetime, rather than when it goes out of scope.  The good news is that at least this bad code won't compile:

```
error[E0499]: cannot borrow `job_queue` as mutable more than once at a time
   --> src\heap.rs:447:21
    |
441 |         let mut handle = job_queue.get_mut(3).unwrap(); // Changed!
    |                          --------- first mutable borrow occurs here
...
447 |         assert_eq!(&job_queue.pop().unwrap().description, "The boss wants this yesterday!");
    |                     ^^^^^^^^^ second mutable borrow occurs here
448 |         // Handle is dropped here, which is too late
449 |     }
    |     - first borrow might be used here, when `handle` is dropped and runs the `Drop` code for type `HeapItemRefMut`
```

The key point is that `handle`'s lifetime is extended to encompass the implicit `Drop` call when `handle` goes out of scope.  This error also points us in the direction of the solution:

```rust
let mut handle = job_queue.get_mut(3).unwrap(); 
*handle = Job {
    id: 3,
    priority: 200,
    description: "The boss wants this yesterday!".to_string()};
drop(handle); // now handle drops here
assert_eq!(&job_queue.pop().unwrap().description, "The boss wants this yesterday!");
```

This works, although it's not as cool as the first example.  We're burdening the client with explicitly dropping `handle`, and with deducing that they *need* to explicitly drop `handle` from the compiler's error message. Nevertheless, it's a big consolation that the compiler will enforce the drop call when it's necessary to prevent dirty reads from our heap.  In some cases, it may even be desirable for the client to delay dropping the handle.  For example, if they want to make several modifications to an object in the heap, it would be more efficient to bind the handle once, do all the modifications, then run the clean-up code just once.  If each modification is done using a different temporary handle, the drop code will run multiple times, which isn't necessary.


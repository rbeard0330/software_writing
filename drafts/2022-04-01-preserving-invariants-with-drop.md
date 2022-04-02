---
layout: post
date: 2022-04-01
title: "Preserving Invariants in Rust with Drop"
---

One solution is to restrict clients' access to stored data to specified operations.  Then the author, knowing exactly what the client may have done, can ensure that any damage to invariants is cleaned up before the operation returns.  As an example, the textbook min-heap often exposes an operation `ReduceKey` that takes a given item and reduces its key (thereby moving the affected item closer to the top of the heap).  Since the data structure knows which object was affected and what the change was, it's relatively straightforward to fix up the heap invariant: all we need to do is invoke our `sift_up` procedure on the targeted item, and the invariant will be restored.

While this solution is simple and hygenic, it is also somewhat limiting.  In particular, what happens if a heap item's key is not an independent property, but rather is derived from the other properties of the object.  For example, imagine that you're sending shipments of bulk commodities from a warehouse, and you want to dispatch the most valuable shipment first.  You could compute the value of each shipment, stash all the shipments into a max-heap, then pop an item off every time it's time to load up a train.  But what happens if rats eat half of a shipment of grain and you need to modify that entry to reflect its reduced value?  If `shipment_value` is a static and independent property of the shipment, then you can just change it.  But what if `shipment_value` is actually computed dynamically from properties `grain_price` and `shipment_size`?  Now we need to do two things: we need to reduce `shipment_size`, then we need to instruct the heap to reduce the sorting key of that shipment to restore the heap invariants.  If we forget either step, we'll end up breaking the heap invariant.  Alternatively, we could write a custom heap structure that exposes a `change_shipment_size` method to do both operations, but that's also unsatisfactory.

## Destructors to the rescue

I recently discovered a rather elegant solution to this problem in Rust.  Consider the following `Heap` struct:

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

This heap stores any item that can produce a numeric key and id on request (or can be converted into such a type).  In addition to implementing the standard operations, we'd like to allow `HeapItem`s to be modified in place.  Exposing a `decrease_key` operation won't work, because we don't know how to modify the key of a generic `HeapItem`.  We could simply provide mutable access to clients: 

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
This isn't entirely terrible, but we've pretty much given up on having real confidence in the correctness of this heap.  It's simply not reasonable to expect that your users will always remember to call the fix-up method after they modify the data.  They *should*, but they just won't, at least not always.

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
        println!("reference dropped--restoring invariants");
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
    job_queue.pop().unwrap();
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

When run:
```
$ cargo test
    Finished test [unoptimized + debuginfo] target(s) in 0.05s
     Running unittests (target\debug\deps\trapper_keeper-72f1867c5dd2762c.exe)
before modification
restoring invariants when reference dropped
after modifications, before read
after read
```

Our fix-up code was automagically inserted right after the data was modified!

---
layout: default
date: 2022-02-17
title: "Linked Lists and Clipper Ships"
---

During a recent read of Knuth's *Art of Computer Programming*, I came across the following interesting note in a discussion of implementing radix sort:

> An alert, "modern" read will note, however, that the whole idea of making digit counts for the storage allocation is tied to old-fashioned ideas about sequential data representation.  We know that *linked* allocation is specifically designed to handle a set of tables of variable size, so it is natural to choose a linked data structure for radix sorting.  Since we traverse each pile serially, all we need is a single link from each item to its successor.  Furthermore, we never need to move the records; we merely adjust the links and proceed merrily down the lists.

This passage is interesting for two reasons:
1. In general, one of the most valuable insights you can gain from reading "older" works is to understand what the future was thought to be before it happened.
2. More specifically, using a linked data structure for anything other than a technical interview is quite contrary to most advice.  (See the hilarious intro to *[Learning Rust with Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/)*.)  Is this the rare use case for a linked list?  Rather than ponder it, let's just try it and see.

### Two Implementations of Radix Sort

#### Radix refresher

If you haven't been brushing up on your algorithms for interviews lately, a refresher on radix sort may be helpful.  Radix sort is a non-comparison sorting algorithm that runs in O(n) time.  Like all linear-time sorting algorithms, radix sort is not fully general, and requires a notion of place value and least-to-most significant digits.

Radix sort works by making a series of passes over the list to be sorted, considering one or several digit positions on each pass, starting with the *least-significant* positions.  The list is partitioned (stably!) into a number of sublists based on the value of the relevant digits of each item.  The sublists are then stitched back together in order, and the process is repeated using the next most significant digit or group of digits.  Though it's not intuitive, at least to my mind, this is a valid sort by construction.  Each item in the output list that has a smaller value in the most significant position comes before each item with a larger value in that position, because the output list was assembled that way in the final pass.  Further, among items with an equivalent value in the most significant position, items with a smaller value in the next most significant position come before items with a larger value, because:
* The list that was produced by the second-to-last pass was assembled to achieve this result, and 
* When that list was partitioned in the final pass of the algorithm, the new sublists were generated *stably*, meaning that the relative order of items in each sublist is the same as in the original list.

With these two facts, we know that if `X` and `Y` are equal in the first `N` positions, and `X` is greater than `Y` in the `N + 1` position, then `Y` will have been placed before `X` in the `N + 1`th-to-last pass of the algorithm, and each subsequent pass will have left their relative positions undisturbed.  This is enough to establish a correct sort.

The central implementation challenge of radix sort is how to store items as they are bucketed in each pass, and then how to rejoin those buckets into the next partially sorted list.  As Knuth indicates, linked lists seems to be an ideal fit.  At the start of each pass, we can initialize each bucket with a `Nil` list, then we can easily append each item to the tail of the appropriate list with a single operation.  Then when all items have been assigned to buckets, we can simply point the tail of each list to the head of the list in the succeeding bucket, and we have a new list.

Linear arrays are apparently less suitable for this task, because we don't know how large each bucket will be.  Thus, we either need to massively overallocate memory and make each bucket large enough to handle the entire list, or else we can start with small allocations at the price of having to copy some buckets over to larger memory allocations when they fill up.  And then generating a complete list requires once again copying each bucket into a new allocation, although it's possible to avoid this step for the intermediate passes.

That's the theory; on to implementation!  (The complete code, including tests and benchmarks that aren't reproduced here, can be found at )

#### Linked list implementation

As with many linked list algorithms, radix sort can be implemented quite elegantly.  First we decide what integer type to use, how many digits to consider in each pass, and then determine how many buckets we'll need:

```rust
const GROUP_DIGITS: usize = 4;
const BUCKET_COUNT: usize = 2_usize.pow(GROUP_DIGITS as u32);
const INT_SIZE: usize = 64;
type Int = u64;
```

Both implementations will use one helper method, which uses bit-fiddling to extract the value at the tested position:
```rust
#[inline(always)]
fn bucket_index_from_value_and_group(group_index: usize, value: Int) -> usize {
    let shift = group_index * GROUP_DIGITS;
    let group_mask = (BUCKET_COUNT - 1) << shift;
    let relevant_digits = value & group_mask as Int;
    (relevant_digits >> shift) as usize
}
```

Our working data structure for this algorithm is essentially a fixed-size array of linked lists.  We call two methods on this structure.  The first is a constructor that takes a single, partially sorted linked list and a group to consider and produces sublists partitioned on that group.  The second method does the opposite, and assembles a new list out of the partitions.  Both are straightforward:

```rust
#[derive(Debug)]
struct LinkedListBuckets([LinkedList<Int>; BUCKET_COUNT]);

impl LinkedListBuckets {
    fn from_iter(it: impl Iterator<Item=Int>, group_index: usize) -> LinkedListBuckets {
        let mut result = LinkedListBuckets(Default::default());
        it.into_iter()
            .for_each(|num| {
                let target_bucket = bucket_index_from_value_and_group(group_index, num);
                result.0[target_bucket].push_back(num)
            });
        result
    }

    fn to_single_list(mut self) -> LinkedList<Int> {
        let mut result: LinkedList<Int> = Default::default();
        self.0.iter_mut()
            .for_each(|sublist| result.append(sublist));
        result
    }
}
```

With this data structure in hand, the actual sort is nearly trivial.  We just call the two methods in turn for each 4-digit group:

```rust
pub fn ll_sort(numbers: &Vec<Int>) -> Vec<Int> {
    (0..INT_SIZE / 4).into_iter()
        .fold(
            numbers.iter().copied().collect(),
            |working_list: LinkedList<Int>, index| {
                LinkedListBuckets::from_iter(working_list.into_iter(), index).to_single_list()
            })
        .iter().copied().collect()
}
```

#### Vector implementation

The `Vec` implementation is slightly more complex, although it follows the same plan.  As before, we have a `Bucket` struct, which is just a fixed-size array of `Vec`s that can be built from an iterator:

```rust
#[derive(Debug)]
struct VecBuckets([Vec<Int>; BUCKET_COUNT]);

impl VecBuckets {
    fn from_iter(it: impl Iterator<Item=Int>, group_index: usize) -> VecBuckets {
        let bucket_size = it.size_hint().0;
        let mut vec_array: [Vec<Int>; BUCKET_COUNT] = Default::default();
        for v in vec_array.iter_mut() {
            v.reserve(bucket_size * 2 / BUCKET_COUNT)
        }
        let mut result = VecBuckets(vec_array);
        it.into_iter()
            .for_each(|num| {
                let target_bucket = bucket_index_from_value_and_group(group_index, num);
                result.0[target_bucket].push(num)
            });
        result
    }
}
```

The `from_iter` method is quite similar, except we pre-allocate some space in each of the `Vec`s to minimize the number of reallocations required later. In the worst case (where all the list items end up in a single bucket), we have excess memory overhead equal to ~2x the size of the list.

The larger structural difference is that we don't have a method for combining our buckets into a new `Vec`.  For a linked list, it's very efficient to stitch the partitioned sublists into a single linked list after each step (this is what linked lists are for!).  Building a new `Vec` each step would be less efficient, since it would require copying the whole list into a new allocation.  To avoid this, we iterate over our `VecBuckets` in place, building each directly from the last.  This entails creating an iterator to manage the process:

```rust
struct VecBucketIter {
    array: [Vec<Int>; BUCKET_COUNT],
    array_ix: usize,
    vec_ix: usize,
}

impl Iterator for VecBucketIter {
    type Item = Int;
    fn next(&mut self) -> Option<Self::Item> {
        if self.array_ix >= BUCKET_COUNT {
            return None;
        }
        while self.vec_ix >= self.array[self.array_ix].len() {
            self.array_ix += 1;
            self.vec_ix = 0;
            if self.array_ix >= BUCKET_COUNT {
                return None;
            }
        }
        let result = self.array[self.array_ix][self.vec_ix];
        self.vec_ix += 1;
        Some(result)
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        if self.array_ix >= BUCKET_COUNT {
            return (0, Some(0));
        }
        let partial_vec_len = self.array[self.array_ix].len() - self.vec_ix;
        let full_vec_len: usize = self.array[(self.array_ix + 1)..].iter().map(Vec::len).sum();
        let remaining = partial_vec_len + full_vec_len;
        (remaining, Some(remaining))
    }
}
```

When we call `into_iter()` on a `VecBuckets`, this structure takes ownership of the array of `Vec`s, and yields the values one-by-one, iterating down each `Vec` in turn.

Again, the actual sort is simple, as our data structures are doing all the work:

```rust
pub fn vec_sort(numbers: &Vec<Int>) -> Vec<Int> {
    (1..INT_SIZE / 4).into_iter()
        .fold(
            VecBuckets::from_iter(numbers.iter().copied(), 0),
            |buckets, index| VecBuckets::from_iter(buckets.into_iter(), index)
        )
        .into_iter().collect()
}
```

### Results

![Bench results]({{"images/clipper_ships/lines.svg" | relative_url}})

Benchmarking the two sorting functions on a variety of list sizes validates the conventional wisdom, with the `Vec` implementation 15-50 times as fast as the `LinkedList`.

| List size | Vec time (time/as multiple of 100k case) | Linked list time (time/as multiple of 100k case) | Linked list as multiple of Vec |
|-----------|------------------------------------------|--------------------------------------------------|--------------------------------|
| 1,000     | 56us / 0.01x                             | 822us / 0.005x                                   | 14.7x                          |
| 10,000    | 354us / 0.06x                            | 12.0ms / 0.07x                                   | 33.9x                          |
| 100,000   | 5.9ms / 1x                               | 173ms / 1x                                       | 29.3x                          |
| 500,000   | 24.7ms / 4.2x                            | 1.19s / 6.9x                                     | 48.2x                          |
| 1,000,000 | 68.7ms / 11.6x                           | 2.66s / 15.4x                                    | 38.7x                          |

This outcome makes sense.  The algorithm does very little work with each number it encounters, just two bit operations.  The rest of the algorithm's work is loading and storing values in the appropriate locations.  The compact and predictable memory layout of a `Vec` allows our hardware to load many values into a single cache line and accurately predict and prefetch the values we'll need in the future.  The linked list values can be located anywhere, so caching provides much less help.  In addition, note that each value is used only once during each pass of the algorithm, so temporal locality is quite low.  In other words, after we use a value, we touch *every other value in the list* before we look at that value again.   

While the array implementation shows the expected linear scaling over these data sizes, the linked list does relatively better at the smallest sizes and especially poorly with the 500k-item list.  This may be related to the nuances of caching.  These benchmarks were run on a machine with 12MB of L3 cache, which is enough to hold as many as 1.5 million numbers (with perfect packing) or as few as 200k (assuming only one 8B integer ends up in each 64B cache line).  If the entire list fits in the cache, we can speculate that all of these values would be loaded into the L3 cache on the algorithm's first pass and remain there for the entire run.  This reduces the penalty for the linked list's inefficient use of cache, because the penalty is a relatively fast L3 cache lookup rather than going all the way to main memory.  As the data set grows beyond the size of the L3 cache, some data will need to be fetched from main memory.[^1]

[^1]: If the cache used a naive policy that evicted the last-recently used item, then *all* of the data would need to be fetched from main memory, due to radix sort's round-robin access pattern.  It appears that modern processors may use more sophisticated policies to avoid this result.  For more information, see [here](https://blog.stuffedcow.net/2013/01/ivb-cache-replacement/) and [here](https://arxiv.org/pdf/1912.09770.pdf).

We can test the hypothesis by benching some larger data sets that are unambiguously larger than the L3 cache. 

| List size | Vec time (time/as multiple of 100k case) | Linked list time (time/as multiple of 100k case) | Linked list as multiple of Vec |
|-----------|------------------------------------------|--------------------------------------------------|--------------------------------|
| 2,000,000 | 152ms / 25.8x                            | 3.33s / 19.3x                                    | 22.0x                          |
| 3,000,000 | 225ms / 38.1x                            | 5.01s / 29.0x                                    | 22.3x                          |
 | 4,000,000 | 283ms / 49.0x                            | 6.94s / 40.1x                                    | 24.5x                          |

While not conclusive, this seems consistent with the hypothesis.  On the larger data sets, the array implementation seems to be regressing towards a 20x advantage over the linked list.  There seem to be four performance regimes:
1. For very small data sets, all data fits in cache, even with the linked list implementation's inefficient usage, and the array implementation has a smaller advantage.
2. For medium-sized data sets, the linked list implementation's data, which is stored less densely, is pushed out to higher levels of the cache, incurring a more substantial performance penalty.
3. For large data sets, some of the linked list implementation's data no longer fits in cache at all, and performance suffers greatly.
4. Finally, for very large data sets, even the efficient array storage pattern overwhelms the available cache, and the array implementation's advantage settles down to a roughly constant ratio reflecting cache prefetching efficiency and the fact that each load from memory brings in more values for the array than the linked list.  


## The Big Picture

That was a lot of work to prove something that we already knew (don't use linked lists).  Why go to all that effort?  Two reasons.  First, despite its technical obsolescence, I have a romantic fondness for the linked list.  It's simple, elegant, and powerful.  Although the low-level mechanics of traversing or modifying a linked list can be finicky, algorithms that use them are often quite beautiful.  Precisely because of its limitations, a linked list has a sort of physicality to it--data that's over *here* is distant from data that's over *there*, both in memory and in program steps.  But if you want to reshape the data by connecting it in different ways, that's easy to do.

The portions of *The Art of Computer Programming* dealing with linked lists put me in mind of the clipper ship.  Both are highly refined evolutions of their respective technologies.  Both are more than physical technologies, but also presuppose a population that has the particular skills and training to use them effectively.  *TAOCP* presents an algorithm that uses quadruply, circularly linked lists to add polynomials.  It's even included in the volume *Fundamental Algorithms* as an exemplar of the different manipulations one does with a linked data structure.  Like crewing a clipper ship, driving a quadruply and circularly linked list is not a task to be approached casually!

What gives the clipper ship and the polynomial-adding algorithm their airs of wistful nostalgia is a combination of two factors.  On the one hand, it is readily apparent that these artifacts represent the cutting edge of their respective fields, sharpened over the course of many previous iterations by skilled craftsmen.  On the other hand, intervening shifts in technology have made these tools largely dead ends, or at least situationally useful at best.  The interplay of historical circumstances and technological capability that gave birth to these particular tools can be clearly seen precisely because they did not spawn new generations of slightly refined variants.  When we aren't distracted by the teleological temptation to see the past as the prologue to our present, we can instead understand it on its own terms.

The critical lesson of both the linked list and the clipper ship is that particular ideas or advancements cannot be evaluated outside their technological contexts.  Both iron and the steam engine existed long before the heyday of the clipper ship, and it's not as if no one thought of building ships of iron or driving them with steam engines.  The *Great Western* steamship was launched in 1838 and began crossing the Atlantic commercially, well before the heyday of the clipper ship in the 1850s.  But in the context of the late 19th century's technology and workforce, sails *were simply better* than steam engines for rapid, long-distance transportation.  Similarly, linked lists *were simply better* than linear arrays in the context of the computers of the 1980s.  Linear arrays are simply better today, for almost all use cases.

Nor was the driving force behind the domination of linear arrays attributable to any particular innovations in array technology or stagnation in linked lists.  Instead, the change was driven by the increasing importance of caching to achieving hardware performance.  Similarly, the decline of the clipper ship is often attributed to the opening of the Suez Canal, which reduced the sailing distance between Europe and Asia and obviated the last few ultra-long-distance routes where clipper ships were still competitive.

While we don't often see well-developed technical disciplines become wholly or largely obsolete due to changes in the technological context, smaller shifts happen all the time.  Moreover, when we experience smaller technological shifts as insiders or participants, it can be easy to overrate the importance of internal causes and underestimate the significance of contextual changes.  It would be easy enough to convince yourself that JavaScript became a dominant language because it's features are well-suited to the needs of front-end scripting, but that would be wrong.  JavaScript rose to prominence when its execution environment was preloaded on every PC and smartphone, and it expanded into other areas because its front-end role created a huge base of talented programmers working in that language.  And on a smaller scale still, most big companies have headaches with "legacy code" platforms.  Usually, these are systems that work perfectly fine on their own terms, but use technologies that have become unfashionable or otherwise obsolescent.  Notoriously, a number of organizations still use COBOL, and one reason for the language's persistence is that it represents non-integer numbers using a fixed-point format that can work better for currency.  The technological context has moved on to IEEE-754 floating point representations, leaving COBOL occupying a precarious, clipper-ship-adjacent niche.
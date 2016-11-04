+++
date = "2016-09-12T22:50:08+02:00"
title = "A Critique of Rust's `std::collections`"
description = "This is an in-depth critique of the bad parts of Rust's collection primitves."
featured = true
share = true
tags = ["rant", "rust", "critique", "data-structures"]
+++

Rust is by far my favorite language, and I am very familiar with it, but there is one aspect that annoys me at times: `std::collections`, a part of the opt-out standard library.

This post will go through the short-fallings of the API and implementation of `std::collections`. I'll try to present alternatives and way to improve it.

**Update**: The title was previously "Why `std::collections` is absolutely horrible". It was in the hope to spark critical discussion, however people were rather annoyed by this title (and I understand why), so I changed it to something less provocative.

# What it contains

`std::collections` has a rather small set of collections (which is a legitimate choice to make to preserve minimality), the catch being that it's an odd choice of collections:

1. B-tree based map and set.
2. Binary heap.
3. Hash table and set.
4. Doubly-linked list.
5. Ring buffer.
6. Random-acess vectors (strictly speaking not in `std::collections` but instead in `std::vec`).

That seems fine, doesn't it? No, it doesn't. If you consider what it lacks of these are very weird choices of structures.

Take binary heap. It is incredibly useful at times, but is it really fit for a standard library with focus on being minimal? Let's look at the statistics:

444 examples of usage of this structure (in Rust) on GitHub. Now, we obviously cannot be sure that this sample is representative, but it should give a pretty good insight on the usage.

Looking through these, approximately 50 of these are tests of `BinaryHeap` itself. Another 50 are reimplementations of it. Around 100 of them are duplicates of other codes (e.g. downloaded libraries). This leaves us with around 250 usages, and that's only slightly more than the incredibly useful `VecMap`, which isn't even in the standard library!

If minimalism really is a goal (which I am going to criticize in a minute), it seems rather weird to have a collection which is barely used more than a non-libstd collection.

Let's move on to doubly-linked list. Searching on GitHub gives you 534 results. I was unable to find _a single place where it was used in a manner that could not be replaced by singly-linked lists_. Chances are that there are some, but they're incredibly rare, and it is odd given that there are no singly-linked list structures in the standard libraries.

**Update**: To clarify here. I'm not arguing that the primitives I propose later deserves a place above these, rather that for a minimal set of collections, the choice is rather odd, given that some collections are even more common than some of these.

# What it doesn't contain

## Concurrent data structures

![How it compares](https://i.imgur.com/GmoylkC.png)

The whole standard library contains exactly two concurrent data structures (note that data structures are different from containers), namely the MPSC-queues (the blocking queue with a limited buffer and the non-blocking with an unlimited buffer). These are used for cross-thread message passing and the alike.

But where are all the other concurrent primitives?

People tend to wrap their structure in `Mutex`, like `Mutex<HashMap<...>>`, but that is often an order of magnitude slower than a concurrent hash table.

Then there's the multithreaded push/pop stacks (as opposed to the queue/unqueue lists), and so on.

There are quite a few implementations of structures as the ones described above, but they are more often than not poorly implemented. The leading library (which has a pretty good implementation quality) is [crossbeam](https://github.com/aturon/crossbeam), but unfortunately it only implements a very limited set of synchronization primitives (no maps, no tables, no skip lists, etc.).

## Singly linked lists

I've already mentioned this, but singly linked list are often useful.

## Keyed priority queues

![An example of a keyed heap](http://www.csit.parkland.edu/~mvanmoer/CSC220/2014/images/heap-9.2.f.jpg)

Keyed priority queues is the structure everyone ask about and looks for, but no one can name it (here's an exercise: go on Google and try to vaguely describe this structure, you will for sure find at least one thread asking for exactly that description, and often no one is able to answer the question or misguidedly proposes binary heaps instead).

Say you have an ordered list of elements, each of which has a priority. Now, you want to be able to retrieve the element with the highest or the lowest priority, with a reasonable performance. Note that mere heaps are not sufficient, since they are not arbitrarily ordered, in the sense that you cannot index them without traversing all elements.

**Update**: People think I'm talking about binary heaps, but they're fundamentally very different. A keyed priority queue is some arbitrarily ordered list (or map) such that elements with high or low priority can be retrieved quickly. Binary heaps are not associative arrays or lists, they do not allow ordering of the elements, and are thus conceptually simpler than keyed priority queues. Note that keyed priority queues are almost always implemented with binary heaps as the backbone. See [this paper]() for an in-depth description of a general-purpose keyed priority queue

Keyed priority queues are used everywhere from cache level regulation to efficient scheduling, and are in my opinion one of the most useful data structures of all.

It is hard to find out exactly how much it is used in the Rust community, given its many names and reimplementations. Only 83 occurrences of the name "PriorityQueue" in Rust code can be found on GitHub, but I suspect the real number to be much higher.

The most famous form of keyed priority queues are Fibonacci heaps, which are used in most major database systems, as well as the Linux scheduler, and many memory allocators.

## Treaps

![Insertion in treaps](http://bluehawk.monmouth.edu/rclayton/web-pages/s10-305-503/treapsf4.png)

Treaps are generally faster than other self-balanced trees (on the average), but the really killer feature is the bulk operations. These are highly efficient algorithms for union, intersections, and set differences.

When the programmer is manipulating sets like this (union, intersections, and so on) and iterators aren't sufficient (i.e., it is for storage, not iteration), these can be incredibly useful as a high-performance data structure.

## Skip lists

![An illustration of skip lists](http://igoro.com/wordpress/wp-content/uploads/2008/07/skiplist.png)

Skip lists are more niche than the structures described above, but they have excellent performance characteristics.

Skip lists are conceptually similar to N-ary trees, but in the representation of a list. They're a probabilistic data structure, which holds a list and some number of sublists such that the _n_'th sublist is a sublist of the _n - 1_'th sublist. Search can be visualized as binary search by observing how two paths can be taken: A) go to the next sublist B) follow the link.

The reason a good implementation outperforms a good implementation of classical binary search trees has to do with two reasons:

1. On average, 50% of the links followed under a search are cache local, whereas B-trees, for example, are around 20% (**update**: The original number said 0%, turns out my tests were wrong) cache local.

2. No tree rotations or equivalent operations are needed during insertion.

Skip lists has its use cases, and a _good implementation_ (flat array, SLOBs, and unrolled lists) can easily outperform B-trees. For really big sets, however, skip lists tends to be slower due to not being as rigidly balanced.

## Self-balancing trees

![An example of a left-leaning red-black tree](https://i.imgur.com/x4QQlZp.png)

As mentioned, Rust's standard library already has an excellent implementation of B-trees, a popular form of self-balancing trees.

The other popular self-balancing trees are good candidates as well (AVL and LLRB/Red-black). While they do essentially the same, they can have very different performance characteristics, and switching can influence the program's performance vastly.

Having a diverse set of such structures can be good, especially if the documentation details which one to use based on your use case.

## SLOBs (aka. pointer lists, memory pools, typed arenas, etc.)

![An example SLOB](https://i.imgur.com/MoaaFPc.png)

This is a very simple, and yet very powerful, data structure. In fact, they are nothing but a glorified singly linked list of pointers to some type. The dealbreaker is the fact that it requires no storage aside from the data it holds.

That is, no allocation is needed to push and pop pointers from this list. This is possible by letting the data which is inactive hold the list itself.

So what's the big deal here? It turns out to be extremely useful for region-based memory management. If you have a lot of allocations of the same type, it is often multiple orders of magnitude faster than allocating each of them seperately, and what's even cooler is the data locality it provides: Since it is based on one big contagious segment broken down into pieces, it will only cover a few pages, and consequently it is cache efficient (this fact will be abused in a minute).

# "We can just leave it to other libraries"

A common talking point is that we can simply outsource it to external libraries. Unfortunately, they cannot provide an essential property of the standard library: standardization. Standard libraries serves for making sure t
he ecosystem is uniform. If something is crucial for keeping the ecosystem together, it deserves a place in the standard library. These are severely underused due to the stigma around adding new dependencies.

Standardization is absolutely crucial for adoptation.

# Criticizing the structures it do have

## `HashMap`

![Open addressing](https://upload.wikimedia.org/wikipedia/commons/thumb/b/bf/Hash_table_5_0_1_1_1_1_0_SP.svg/380px-Hash_table_5_0_1_1_1_1_0_SP.svg.png)

Rust's hash table implementation is perhaps the criticized part, not because it is a bad implementation, but because it is a very performance-critical component, and yet has its rough edges (including a, for some, odd choice of hash functions).

**Update**: The code is very well-written. The best way to study it is reading it. I recommend doing that.

A quick overview of the Rust `HashMap`/`HashSet` implementation is:

- Open addressing (Robin Hood hashing)
- 90.9% load factor before reallocation
- Defaults to Sip-hasher (cryptographic hash function)

Let's just go through these one-by-one and see what's wrong:

### Robin Hood hashing

Robin Hood hashing is a double-hashing variant quite, in which you rehash until the slot is free. Robin Hood hashing improves plain double-hashing by making sure the slots occupants are ordered by the probe length.

So what's the problem here? Well, the cache efficiency is not exactly ideal, but we get freedom from clustering in return.

The "opposite" approach is linear probing (where you add some constant - often 1 - to the slot number until it is free), which has the opposite nature: Cache efficiency is really good, but it is very sensitive to clustering.

A reasonable alternative which takes the best of each of these solutions is quadratic probing, which simply uses a quadratic polynomial to jump between slots (or, in analogy to the one given above, the constant in question increases linearly).

In most scenarios (especially for large tables), quadratic probing has fewer cache misses, due to better data locality.

There's no reason to have strong opinions on this subject. The difference is rather small, but interesting nonetheless.

**Update**: As people pointed out, this section contains the error of confusing Rust's implementation of Robin Hood hashing with the original paper. In fact, it is very different: Rust's implementation doesn't rehash, but instead linearly probes, that simply makes the difference even smaller, and it makes it a pretty good choice. I don't know enough to be able to judge if it is better or worse than quadratic probing.

### A high reallocation threshold

This mostly comes down to a trade-off between between memory and CPU. If you think about it, 1:9 empty slots is a pretty dense table with an average probe length of 6 rehashes. Potentially (for very large tables) that can lead to 3-6 (avg. 3 or 4, depending on which test) cache misses for just a single lookup.

The advantage is that it is relatively memory efficient, but that should really only be a concern for really big tables. For most tables, this is way too high, and it trades CPU cycles for memory (which is almost unimportant these days).

I personally think that having a constant factor is a bad idea. I think it should be some function of the number of elements in the table, such that the factor is lower for small tables (where memory isn't a concern). This way you get memory efficiency when it matters.

### Sip-hasher

![A round of Sip](https://i.imgur.com/smjww32.png)

Sip-hasher is a cryptographic hash function, and as with most cryptographic hash functions, it is slower than the non-cryptographic counterpart.

**Update**: Some people have pointed out that it is in fact not a cryptographic hash functions, but rather an efficient and yet sufficiently secure hash keyed function. In contrary to DJB2 or xxHash, it is not easy to invert/generate a preimage, which has the consequence of being hard to generate collisions on purpose, a property that can be important for publicly exposed structures (such as databases, KV stores for servers, etc.).

And it doesn't even have a measurable better quality. I tried giving three different hash functions various data sets. Collision-wise they did equally on every data set, with exception of the English dictionary (as seen below). In every single test, my home-made hash function "long hasher" beats sip-hasher on performance _by a significant factor_ (around 30%)...

    ~ SipHasher
        Filled buckets: 2048
        Max bucket: 245
        Time: 0 s. 39232 ms.
        GB/s: 0.8565382742517557
    ~ DJB2
        Filled buckets: 2048
        Max bucket: 238
        Time: 0 s. 39463 ms.
        GB/s: 0.784737479297558
    ~ LongHasher
        Filled buckets: 2048
        Max bucket: 239
        Time: 0 s. 29562 ms.
        GB/s: 4.95375004585191

Some investigation shows that the vast majority (98%) of the time of retrieval is used on hashing (it's not clear if the same is true for insertions, but it still have a major influence)). Say you used hash maps for caching database queries. That could potentially translate to 30% faster retrieval on cached queries.

My point isn't that LongHasher is fantastic, but my point is that there are hash functions which vastly beats sip hasher performance-wise.

On a side note: These numbers are quite impressive, and if you don't believe me [you can run it yourself](https://gist.github.com/anonymous/3b0b489137af9006d5c498f10d42514a). The reason that long hasher is able to outperform them both is that it consumes eight bytes at once. Otherwise, it is really just multiplying by a prime, adding some constant, multiplying by another byte, rotating right and then XORing by some key.

Now, if it isn't quality, then what's the reason for using a cryptographic hash function? One reason often cited is Denial-of-Service resistance, and that's a valid concern, but is it really something that everyone should pay for?

You may know the famous "What you don't use, you don't pay for" idiom. This is a core part of the "abstraction without overhead" principle. It is relatively rare to actually need DoS-protection, and you pay for this whenever you use `HashMap` without overwriting the hash function.

And, ignoring that point for a moment, The idea that your code is 'secure by default' is a dangerous one and promotes ignorance about security. You code is _not_ secure by default.

If you really do need a fast and yet secure hash function, Sip-hasher is a wonderful choice. To be clear, I'm not arguing against Sip-hasher (I actually like the function), but rather against having it as a default choice.

## `BTreeMap`

`BTreeMap` and `BTreeSet` are generally a good implementation. My only criticism has to do with cache efficiency, which is relatively bad when compared to other implementations (Java, mainly). In fact, around 60% (**update**: Some have obtained other results, ranging from 10%-60%.) of the links followed leads to cache misses. For a map 1000 elements, a lookup would result in approximately 6 cache misses. For 10000, the number is 8.

These can be quite expensive. A solution is proposed in the section about "Cache-efficient structures".

## `VecDeque`

`VecDeque` is a decent implementation. The only problem is that you cannot range index (slice) it. This is due to the very nature of ring buffers.

An alternative to conventional ring buffers is [biparite buffers](http://www.codeproject.com/Articles/3479/The-Bip-Buffer-The-Circular-Buffer-with-a-Twist), which has essentially the same performance, but allows this (and other interesting) API.

## MPSC

MPSC is a popular tool for message passing, but a critical point is often overlooked: Every queuing/dequeueing involves a malloc call. That sounds pretty bad, doesn't it?

MPSC is supposed to be lock-less, but that isn't the case if the tcache is empty. Then it involves a lock.

Since you effectively queue and dequeue all of the time, you actually waste allocations going in and out the allocator. That's a major overhead, and totally unreflected in the API, giving an illusion of zero-cost.

And, it turns out that it isn't necessary. Because of the ring-buffer-like structure of MPSC, you can effectively store it all in a concurrent SLOB list, making malloc calls incredibly rare (ideally only upon the first queue).

My benchmarks I've made on [ralloc](https://github.com/redox-os/ralloc), a memory allocator I wrote, (which uses mpsc internally for cross-thread frees) shows a significant performance gain. An exercise for the reader is to do the same for Servo and try to see if it affects performance.

## `Vec`

My criticism of `Vec` is the lack of API for manual management. One particular missing thing is the ability to replace the reallocation strategy with a custom one.

Vectors are used everywhere and they often have different usage patterns, many of which can be exploited to improve performance and memory efficiency.

Another thing I sometimes need is the ability to get a mutable reference to the element I pushed without needing extra bound checks (note that in most cases this is a trivial optimization for LLVM, but it adds a lot of convenience). This could simply be solved by having `push` return `&mut T`. This is technically a breaking change but I doubt it will affect anybody.

# Cache-efficient structures

I spoke a little about SLOB-based arenas previously. It turns out to have major impact on the cache efficiency, and thereby the performance, of the structure.

The idea is that each structure holds an arena which only spans a few memory pages, ensure data locality. Obviously, this is more memory hunky, but it is conceptually similar to vectors which reserve extra memory to avoid reallocation. In this case, we are looking for avoiding allocation instead.

Depending on what you're doing it affects the performance positively by 3-10% (B-trees), 5-15% (linked lists), or 40-80% (mpsc). Those numbers are quite impressive (especially the last one).

# Replacing the allocator

Another lacking feature is an `Allocator` trait, which is intended to be the bound of some generic parameter in all the collections, allowing you to replace the allocator to exploit allocation patterns of the structure.

An RFC for exactly this [already exists](https://github.com/rust-lang/rfcs/pull/1398) and is merged, but the implementation is incomplete.

# Hiding `Box`

An unfortunate thing is hiding the overhead by letting the function itself allocate, instead of letting the caller do it. This is (or at least, should be) considered bad style, because the API ought to reflect the semantics and performance characteristics. If the allocation is hidden to the programmer, she might not realize the expensive operations behind the scenes.

# The good parts

The implementations them self are really good and well-tested, and many of the points I made above are only relevant, when you are looking for very fine-grained performance. I have not criticized the API itself, because I think it does very well. Rust's collection API is one of the most well-designed I've seen.

It is also worth noting that the ecosystem contains lots of wonderful structures and implementations of these. Rust's ecosystem is getting more mature by every day, and to this day, it even contains very niche structures for very specific purposes, and that's really great since it means expansion of Rust's domain.

# Conclusion

Rust's "collection of collections" has quite a few short-fallings. Fortunately, Most of the problems described above are not inherent, and can be fixed. The first step through fixing a problem is diagnosing it, and I hope that this post is able to initiate some critical discussion around the implementation, API, and choice of collections for the standard library.

There are already many different proposals floating around in the community, as well as implementations and so on. I hope we can look into lifting these or replacing them in libstd.

The [contain-rs](https://github.com/contain-rs) GitHub organization does a great job at providing a collection of various data structures, most of which are incredibly well-written. It would be interesting to see some of those in the standard library.

# An apology

The original title was exaggerated and inflammatory. To clarify, I don't think it is horrible, nor even bad, rather I'd want a discussion/critical examination of the module and ways to improve it. I do realize that many people have spend a lot of time on implementing all these, and I admire it, calling it horrible (even if it is only in the title) is not fair, and it will likely distract the reader from the message of the post instead of inciting discussion. The mistake was entirely mine, and I should have given it a better title. Sorry.

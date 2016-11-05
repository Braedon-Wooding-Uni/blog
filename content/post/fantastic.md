+++
author = "Ticki"
comments = true
date = "2016-09-14T16:07:34+02:00"
description = "Now that we've seen the bad stuff of `std::collections`, we're going to look into all the good stuff and various techniques used to optimize and improve the collections"
draft = false
cover = "https://news.uic.edu/files/2014/11/649328_45676950.jpg"
menu = ""
share = true
slug = "fantastic"
tags = ["rust", "data-structures", "standard-library", "follow-up"]
title = "Why Rust's `std::collections` is absolutely fantastic"

+++

My [last blog post](http://ticki.github.io/blog/horrible/) was about all the short-fallings and problems `std::collections` has. This post will be about the opposite: all the good things about `std::collections` and what other languages can learn from Rust.

This post is a part of an on-going series of posts criticizing and praising various parts of Rust.

# The philosophy of `std::collections`

Rust has an intentionally small set of collections. This has both advantages and disadvantages.

For one, the surface area becomes smaller, which allows libstd to focus on the support and performance of a few really good collections.

Another advantage is the fact that the user can easier choose which collection. If you take Java's standard library, which is arguably bloated, the programmer is often confused about which data structure to use. Rust's small sets of primitives is easier to work with.

If a specialized data structure really is needed, [crates.io](https://crates.io) contains many cool implementations of various data structures, including some _really_ specialized ones. That just proves that the Rust ecosystem is healthy.

# When is a primitive appropriate for the standard library?

There are generally two factors to consider:

1. Standardization: The standard library serves for consituting some uniformity between libraries. This includes (but is not limited to) API interoperability, familiar API.

2. Convenience: Many primitives in the standard library are there simply for the sake of convenience.

Rust's standard library is the smallest set of primitives which are sufficient per the factors above. The collection module is no different: The smallest viable set of collections for standardization and convenience is provided.

I think the `std::collections` accomplishes that pretty well. It contains a few, frequently used structures, such as hash tables, B-trees, and vectors.

# `Vec<T>`

![Vector growth](https://i.imgur.com/UDeQOFK.png)

Most programmers are familiar with vectors, the growable contiguous array with random-access property. Indeed, it is a very simple and well-defined structure. It is not a particularly elegant structure, but it is practical, simple, and fast.

The only thing that is really implementation defined is the reallocation strategy. Vectors tries to avoid excessive allocations by allocating some extra capacity, making pop/push amortized O(1). I think the choice of strategy is a sane one. It's relatively simple: The initial allocation is some constant number of elements, when the capacity is reached, multiply it by two.

The API is very similar to C++'s `std::vector`. Rust has no overloaded syntax with respect to initializing it, however it provides macro `vec!`, which allows for doing exactly that.

`Vec::new()` does not do an initial allocation, this is based off the observation that many vectors are empty throughout their life. This mean that we can (without calling `malloc`) initialize an empty vector with capacity zero. This will simply set the start point to 0x1 (the reason it isn't null is because Rust has a language-supported optimization, which makes type marked as non-null nullable in certain tagged unions).

## Side note: Rust's allocation API

![A simple diagram comparing the two](http://i.imgur.com/cH5a8gu.png)

One of my favorite aspects of Rust is that it provides a malloc API which has actually been thought about, in contrast to most other APIs (I'm looking at you libc `malloc`).

There are two main differences from `malloc`:

1. The programmer provides the size. Instead of doing `free(pointer)`, you do `free(pointer, size)`. That might seem like a minor change (and maybe even annoying), but the motivation is pretty clear: Due to the limitations of the libc `malloc` API, most allocators add a the size to the block metadata of the allocation. This ranges from 2-8 bytes¹, but if you think about it, that's quite a lot for small allocations¹. The information is redundant, because the user almost always know the size of the buffer. For example, `Vec` would only have to provide its capacity, etc.

2. You can force inplace allocations (buffer extensions). In the libc API, there is no way to reallocate inplace, if possible. You have to go through `realloc`, which might call `memcpy`. Considering that inplace reallocation is a lot faster, there are scenarios where you'd like a reallocation, but it is not important enough for you to pay the `memcpy`. Take `Vec` for example, if it is "almost at capacity", you'd like an extension, because waiting until it is filled up might mean that said memory is already taken. On the other hand, doing an early reallocation is bad style, since it provides no real benefit over doing it when the capacity is full.

It is more complex than libc's `malloc` API, but Rust programmers rarely use these symbols directly. Rather, it is abstracted away under safe primitives. The implementations of such primitives might exploit it for performance.

Now, this is only an API, and Rust currently uses Jemalloc, so not all of these features are supported. I'll take the liberty to shamelessly plug [ralloc](https://github.com/redox-os/ralloc), a pure Rust memory allocator making full use of the Rust allocation API.

¹ Note that various optimizations (such as seperated segments and specialized bookkeeping for small blocks) can be done (especially for small allocations), but you ultimatively cannot remove the cost without changing the API.

# `HashMap`

## Collision resolution

![Open addressing](https://upload.wikimedia.org/wikipedia/commons/thumb/b/bf/Hash_table_5_0_1_1_1_1_0_SP.svg/380px-Hash_table_5_0_1_1_1_1_0_SP.svg.png)

In [the last post](http://ticki.github.io/blog/horrible/), I strongly criticized the hash table implementation, but after extensive discussion on Reddit and IRC, I changed my mind. The only thing I really disagree with for `HashMap` is the choice of Sip-hasher.

My analysis of the implementations had one big flaw: not differentiating between linear Robin Hood hashing and double Robin Hood hashing. Rust is using the former. My claim that it had a lot of cache misses was completely wrong: It actually have excellent cache behavior.

Due to the terrible naming, it can be very confusing (even [it's Wikipedia page](https://en.wikipedia.org/wiki/Hash_table#Robin_Hood_hashing) is wrong), but there are two variants: The classical, double-hashing variant (which was described in Pedro's original paper), and the modern linear probing variant. I wrongly said that Rust is using the former, but there are good news: It doesn't!

What Robin Hood does is minimizing the probe lengths by giving up the slot if the probe length is longer. In other words, the worst-case is improved and it is more equally distributed making clustering less damaging.

Let's consider a table like this:

        [0|neat|feet]
        [1|good|hood]
        [0|load|boat]
        [ |    |    ]
        ...

Then let's say we want to insert `fool` at `cool`. Let's say that `hash("cool") == 1`:

        [0|neat|feet]
       *[1|good|hood]
        [0|load|boat]
        [ |    |    ]
        ...

As you can see the entry marked with `*` is already taken, so we need to probe forward, but that's taken too. Well, right now we have probed one time, so our probe length is one, compared to the probe length of the `load` slot, which is zero. To minimize our probe length, we replace `load` with `cool`:

        [0|neat|feet]
        [1|good|hood]
        [1|cool|fool]
        [ |    |    ]
        ...

We still need to find a place for our old entry:

        [0|load|boat]

Luckily, the next slot is empty, so we get:

        [0|neat|feet]
        [1|good|hood]
        [1|cool|fool]
        [1|load|boat]
        ...

If we had simply done linear probing, we'd have:

        [0|neat|feet]
        [1|good|hood]
        [0|load|boat]
        [2|cool|fool]
        ...

Then looking up `cool` would need two jumps, i.e. a bad worst case.

In other words, you save the slowness of rehashing/cache missing, and yet preserve the cool features of Robin Hood hashing.

I think it's a good choice.

## `SipHasher`

I'm still very strongly opposed to having `SipHasher` as default hash function, but it is not entirely without advantages. Security is one.

What I really want to talk about is a sane compromise between `SipHasher` and a fast hashing functions which ensures the same security/DoS-resistance properties of `SipHasher`, while having better performance in most cases. This is not yet implemented, but it has been floating around in the community for a while now, and it is definitely a plausible alternative, which we will likely see in Rust in the future.

The proposal is called _adaptive hashing_, and it is in fact a relatively simple strategy: Use a fast hash function initially, and when a probe length in the table surpasses some threshold, switch to `SipHasher`.

A long probe sequence is relatively rare to occur naturally, so it is a good indicator for exploitation.

A more complicated version of the proposal is to make it slot-specific, in such a way that the switch only happens when a specific probe length exceeds some threshold.

An initial implementation (outside the standard library) along with benchmarks can be found [here](https://github.com/contain-rs/hashmap2/pull/5).

# B-trees

![An example B-tree.](https://i.imgur.com/tUmeDcr.png)

B-trees are one of those structures that people always implement wrongly in one way or another. Fortunately, Rust's standard library gets this one right.

The code is clear and the performance is good. The only catch is the cache efficiency, which there are still room for improvement of.

# Binary heap

![A binary tree satisfying the heap property.](https://upload.wikimedia.org/wikipedia/commons/thumb/1/1c/Heap_delete_step0.svg/500px-Heap_delete_step0.svg.png)

`std::collections::BinaryHeap` is a fairly standard implementations of a flat-array binary heap. Only one thing here really stand out: the `Hole` struct.

The definition looks like this:

```rust
struct Hole<'a, T: 'a> {
    data: &'a mut [T],
    /// `elt` is always `Some` from new until drop.
    elt: Option<T>,
    pos: usize,
}
```

It represents some segment from the heap array which is invalid (i.e. shouldn't be read). This is enforced through the static checking of rustc, which verifies that mutable references are mutually exclusive.

I like this approach over the more standard ones, due to eliminating many bugs you easily encounter while writing a flat binary heap.

# Code quality

The code quality of `std::collections`, or `std` is general, is very good. It contains a lot of comments explaining optimizations implementation strategies and other things important for understanding the code:

|       Files |      Total |     Blanks |   Comments |       Code |
|------------:|-----------:|-----------:|-----------:|-----------:|
|          18 |      18868 |       1530 |       8567 |       8771 |

Here's an example:


    // > Why a load factor of approximately 90%?
    //
    // In general, all the distances to initial buckets will converge on the mean.
    // At a load factor of α, the odds of finding the target bucket after k
    // probes is approximately 1-α^k. If we set this equal to 50% (since we converge
    // on the mean) and set k=8 (64-byte cache line / 8-byte hash), α=0.92. I round
    // this down to make the math easier on the CPU and avoid its FPU.
    // Since on average we start the probing in the middle of a cache line, this
    // strategy pulls in two cache lines of hashes on every lookup. I think that's
    // pretty good, but if you want to trade off some space, it could go down to one
    // cache line on average with an α of 0.84.
    //
    // > Wait, what? Where did you get 1-α^k from?
    //
    // On the first probe, your odds of a collision with an existing element is α.
    // The odds of doing this twice in a row is approximately α^2. For three times,
    // α^3, etc. Therefore, the odds of colliding k times is α^k. The odds of NOT
    // colliding after k tries is 1-α^k.
    //
    // [...]

Such comments can be found in many places and provides great insight into the small tricks used for speeding things up.

# Conclusion

We've seen a few insights and techniques used, as well as the reasoning behind it and ways to improve it. It's not horrible (or even bad) as a whole. It has its rough edges (some of which were pointed out in the last post), but it's in constant improvement and has many great primitives.

I'd like to thank all the people who've implemented these collections, as well as the on going out-of-tree implementations of collections.

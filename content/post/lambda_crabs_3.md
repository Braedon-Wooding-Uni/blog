+++
date = "2016-06-08T11:24:24+02:00"
title = "Lambda crabs (part 3): Region-based alias analysis"
description = "In this post, we show how you can analyse alias properties through region data."
tags = ["lambda-crabs", "rust", "mathematics", "pointers"]
+++

In the [last post](http://ticki.github.io/blog/lambda_crabs_2/), we saw how to
infer regions and their span. In this post, we will cover aliasing and how to
ensure guarantees through region analysis.

## Aliasing, mutable aliasing, and unsafety.

Two pointers are said to be _aliased_, if they refer to the same object. Alias
analysis is essential to program verification, optimizers, and compiler theory.

Alias analysis is the study of which pointers are aliased and, more
importantly, which pointers _aren't aliased_.

Rust guarantees that no mutable reference is aliased. This is statically
checked, and we will show how in this post.

So, why is aliasing guarantees even needed?

The answer is that I need to be able to reason about the invariants of the
pointers content, while being sure that those aren't broken in the period of
accessibility.

Furthermore, we want strict thread-safety, which requires guarantees about
shared mutable state.

## Different values, different namespaces

To reason about mutability overlaps and aliasing through regions, we need a
notion of different namespaces.

For example, say some variable X is referenced in a lifetime `'a`. Does that
mean another variable Y living in the same scope can't be mutated?

Of course not! Let's consider:

```rust
{
    let mut a = 2; // ------+ 'a
    let b = &a;    // ----+ | 'b
    let mut c = 0; // --+ | | 'c
    c = 1;         //   | | |
    c = 2;         //   | | |
} // -------------------+-+-+
```

As you can see `a` is aliased, thus mutating it is not allowed. However, `'a:
'c`, but that doesn't mean they refer to the same.

Holding a global namespace would make the above example fail, since it has no
distinction between `a` and `c` and their respective lifetimes.

For that reason, we need to segregate the regions, such that we can effectively
reason about aliasing without mixing values up.

## Sublattices

We talked a bit about lattices and their applications in the last part. I
recommend reading that if you do not know what a lattice is.

Now, let's introduce the notion of a _sublattice_:

_M_ is a sublattice of _L_, if _M_ is a nonempty subset of _L_ forming a
lattice under _L_'s meet and join operators.

Take a lattice,

```
       Join(a, b, c)
           /\
          /  \
         /    \
   Join(a, b)  \c
       /\      /
      /  \    /
     /    \  /
   a/      \/b
    \      /
     \    /
      \  /
       \/
    Meet(a, b)
```

(by the way, this is why it is called a lattice)

then

```
   Join(a, b)
       /\
      /  \
     /    \
  a /      \b
    \      /
     \    /
      \  /
       \/
    Meet(a, b)
```

is a sublattice, since it holds all the conditions:

1. It is a nonempty subset.

2. It shares meet and join, while preserving closure (you can easily check this
   yourself).

## Region classes

Region classes has many names, but none which is universally agreed upon, I
prefer the name region classes. As we say, a rose by any other name would still
smell as sweet.

Let _L_ be our region lattice, define a _region class_ of _L_ as a sublattice
of _L_, in the context of segregating regions.

In particular, assign each value a region class. Say the value has the bounds
(outlives) `{a, b, c, d...}`, then we derive our region class as the cyclic
sublattice, `<a, b, c, d...>`. In particular, this means _the smallest
extension which forms a sublattice of L_.

The compiler keeps a log of the region class of every variable. This is then
used for alias analysis:

## Pointers and references

Taking an immutable reference extends our region class to contain the region of
this particular reference, denoted `M[N]`.

Mutable references, on the other hand, works slightly different. The region
class and the region of the reference _must be disjoint_, unless we get shared
mutability. With this requirement satisfied, we can proceed to extend the
region class with the new region.

## Mutating a local variable

You may ask, "Can you mutate a local variable while it is borrowed?", the
answer is, "No, you cannot". The reason is the same for the mutable aliasing:
it introduce shared mutable state.

But, how do we handle such mutations?

We introduced `empty(x)`, the empty region at `x`, in the earlier blog posts.
And we can use this to interpret local mutations as well: as taking a mutable
reference for region `empty(x)` and simply mutate it through the reference.

## Applying this method

If we get back to our example,

```rust
{
    let mut a = 2; // ------+ 'a
    let b = &a;    // ----+ | 'b
    let mut c = 0; // --+ | | 'c
    c = 1;         //   | | |
    c = 2;         //   | | |
} // -------------------+-+-+
```

We can see that the region class of `'a` is an extension of `'b`, but `'c` is
not entangled with `'a`'s region class. In particular, `'c` and `'a` belong to
different namespaces and thus, there is no shared mutability.

## Region classes and their relations

A natural question that arise is: Why don't we do region inference seperately
for each region class?

The answer is that distinct region classes are far from unrelated. Each region
class simply defines a value and its aliases, but that doesn't make it isolated
for the rest of _L_.

If you look at our example above, you may notice that `'a` outlives `'c`,
despite being associated with a different region class.

## Questions and errata

Ping me at #rust in Mozilla IRC.

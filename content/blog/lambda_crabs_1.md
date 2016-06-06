+++
date = "2016-06-06T09:12:56+02:00"
title = "Lambda crabs (part 1): A mathematical introduction to lifetimes and regions"

+++

This post will cover lifetimes and regions in depth, with a focus on the mathematical background of regions. That is, what is a region? What rules do they follow? How does the compiler handle them? And how are they inferred?

## Regions and their ordering

So, let's briefly investigate what a region is. A region (or in Rust lingo, a lifetime) is a span of some form, e.g. the token stream. Regions have an outlive relation defined on them.

A region `'a` outlives `'b` if `'b`'s span is covered by `'a`. For example:

```
'a: I---------I
'b: I----------------I
```

As you can see `'a: 'b` since the second span covers the first. But what is the nature of the outlives relation?

## Regions: a poset

One could mistakenly believe that regions are ordered over their outlives relation. An totally ordered set A under ≤ means that any elements a, b ∈ A satisfy all of the following statements:

1. If a ≤ b and b ≤ a are both satisfied, a = b.

2. If a ≤ b and b ≤ c are both satisfied, a ≤ c.

3. At least one of a ≤ b and b ≤ a is true.

To see why the outlives relation is not a total order over the set of regions, consider the case:

```
'a: I---------I
'b:    I------------I
```

The third condition is not met here: neither `'a: 'b` or `'b: 'a` is true.

It turns out that weakening the last condition to only consider reflexivity gives us a structure, that L (the set of regions) classifies. Replace 3. by a ≤ a, and you get a partially ordered set, or a poset.

## Outlive relation as a partial order

So, let's briefly explain how the rules of outliving mirrors the rules of partial orders.

The first rule, the rule of antisymmetry, reads

```
'a: 'b
'b: 'a
-------
'a = 'b
```

So if two regions (lifetimes, borrows, scopes, etc.) outlives each other symmetrically ('a: 'b and 'b: 'a), they are, in fact, the same.

The second rule, the rule of transitivity, is crucial to understanding the semantics of regions:

```
'a: 'b
'b: 'c
------
'a: 'c
```

In other words, regions are hierarchal. It might seem very simple, but the implications are in fact very important: it allows us to conclude things from transitivity. Think of it like you can "inherit" bounds from outliving regions.

For example,

```
'a:      I---------I
'b:   I----------------I
'c: I--------------------I
```

Say we know that, `'a: 'b`, and `'b: 'c`. We can then conclude that `'a: 'c`.

The last rule simply states that 'a outlives itself. This might seem counterintuitive due to the odd terminology, but think of outlives as "outlives or equals to".

In fact, there is only one more thing we know about regions: they have an unique maximal extrema, which outlives all other regions, `'static`. Namely, `'static` outlives any region, `'a`.

And that's all the "axioms" of lifetimes.

## What subtyping is

Before we go to next section, we will just have to briefly cover subtyping. τ is said to be a subtype of υ (denoted `τ <: υ`), if a _type mismatch_, such that τ is inferred to be of type υ, makes the value of type τ coerce into a value of type υ.

In other words, you can replace your subtype by a supertype (the parent type) without getting a type mismatch error.

## Regions are just types: Outlive relation as a subtyping rule

If you think about it, you may notice that lifetimes are used in type positions _a lot_. This is no coincidence, since _regions are just types with a subtyping relation_, which is the very reason you are allowed to do e.g. `MyStruct<'a>`.

In fact, the outlive relation defines a subtyping rule. That is, you can always "shrink" a region span. Let _c_ be a type constructor, 'a → *, then 'a: 'b implies that `'a <: 'b`, that is `'a` can coerce into `'b`.

For example, `&'static str` can coerce to any `&'a str`, since `'static` outlives any lifetime.

Due to the implementation, there are a few limits, though. You can for example not do `let a: 'a` which would be useless anyways.

Syntactically, there is a confusion: lifetimes appears in certain trait places, especially in _trait bounds_. But, in fact, that is only a syntactic sugar for an imaginary trait, let's call it `Scope`, which takes a lifetime.

This represents the scope of a type, so when writing `fn my_func::<T: 'static>()` you can think of it as writing `fn my_func::<T: Scope<'static>>`.

Due to the coercion rules (which will be covered in a future post), this means that if `T: Scope<'a>` and `U: Scope<'b>` with `'a: 'b`, then `T` is a subtype of `U`.

## Inferring regions.

This is the exciting part. Rust has region inference, allowing it to infer the lifetimes in your program.

Due to Rust's aliasing guarantees, it tries to _minimize_ the region's span, while still satisfying the conditions (outlives relations) given.

So, this is just a classical optimization problem:

```
minimize    'a
subject to  A, B, C...
```

A, B, C... are outlives relations. 'a may or may not be free in those.

We will cover how we actually solve this optimization problem in a future blog post, but until then you can see if you can find an algorithm to do so ;).

## Questions and errata

Ping me at #rust in Mozilla IRC.

## Credits

Credits to Yaniel on IRC for the idea for the name of this series. It is based on the famous "lambda cats" series, but since Ferris, the crab, is our Rust mascot, we do lambda crabs, instead.

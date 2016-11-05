+++
date = "2016-06-06T11:06:21+02:00"
title = "Lambda crabs (part 2): Region inference is (not) magic."
description = "I go through the solution to our optimization problem presented in last post. This provides an full algorithm for inference of regions."
tags = ["lambda-crabs", "rust", "mathematics", "regions", "type-inference", "type-systems"]
+++

This post will cover region (lifetime) inference with a mathematical and type
theoretical focus.

## The problem

Inference is a very handy concept. We no longer have to annotate redundant
types, which is a major pain point in languages, that lacks of type inference.

Now, we want such an inference scheme for regions as well.

We described the problem of region inference in [last post](/blog/lambda_crabs_1/) as:

> So, this is just a classical optimization problem:

  ```
  minimize    'a
  subject to  A, B, C...

  A, B, C… are outlives relations. ‘a may or may not be free in those.
  ```

Namely, we want to minimize some lifetimes, while holding some conditions.

## "Adding" regions

One thing we will use throughout the region inference algorithm is the notion of "adding" regions.

You may have seen `'a + 'b` before. Intuitively, `'a: 'b + 'c` is equivalent to
`'a: 'b, 'a: 'c`, but we can go further and use `'a + 'b` as a way to construct
new regions:

Define `'a + 'b` as the smallest region that outlives both `'a` and `'b`.

In a sense, you "widen" the region until it covers both regions:

```
'a:       I---------I
'b:            I------------I
'a + 'b:  I-----------------I
```

## Funky but useless: Regions under addition as an abelian semigroup

A semigroup is an algebraic structure satisfying two properties:

1. Closure, for any a, b in S, a + b is contained in S.
2. Associativity, for any a, b, and c in S, (a + b) + c = a + (b + c).

But in contrary to monoids, there is no identity element.

"Abelian" means commutative. That is, a + b = b + a.

And, in fact, regions follows all these rules, making it an abelian semigroup.

We know two additional facts about our operator:

1. It follows from the fact `'a: 'a`, that a + a = a
2. It follows from the fact `'static: 'a` for all `'a`, that `∃s∈L  ∀a∈L  s + a = s`.

## Regions as a lattice

It makes much more sense to think of regions as a lattice. A lattice is a poset
with two operators defined on it:

1. Join, an unique supremum (that is, least upper-bound). This is our `+`
   operator.

2. Meet, an unique infimum (that is, greatest lower-bound). This isn't very
   useful for the matter of regions, but it is still defined on them.

which follows a set of laws:

1. The law of commutativity: Both meet and join are commutative operators.

2. The law of associativity: Both meet and join are associative operators.

3. The law of absorption: Meet(a, Meet(a, b)) = Meet(a, b) and Join(a, Join(a, b)) = Join(a, b).

In fact, this describes our structure perfectly. In particular, L is an
_upper-bounded lattice_, i.e. we have a maximal element (`'static`).

Lattice theory, which we will cover in-depth in a later post is perfect for
studying subtyping relations.

## Directed Acyclic Graphs

A directed acyclic graph is a finite directed graph with no directed cycles.
That is, any arbitrary directed walk in the graph will "end" at some point.

![](https://upload.wikimedia.org/wikipedia/commons/6/61/Polytree.svg)

Let's forget the `'a: 'a` case for a moment. As such, the regions under our
_strict_ outlive relation, _<_, forms a directed acyclic graph (DAG).

In particular, if two node are connected, with a directed edge A → B, A
represents a region, which _outlives_ B.

Consider we take a reference `&'b T` where `T: 'a`

```
 'static
 |
 v
'a <---------\
 |           |
 |           |
 |           |
 v           |
'b <------- 'a + 'b
```

## Handling cycles

Every lifetime outlives itself, as explained in the last post. So our outlives
relation doesn't form a DAG, due to these cycles.

The solution is relatively simple, though.

Let `{'a, 'b, 'c, ...}` be cycle such that `'a < 'b < 'c ... < 'a`. Due to
transitivity and antisymmetry, we can assume that `'a = 'b = 'c = ...`, thus we
can, without loss of generality, collapse the cycle into a single node.

This lets us interpret the graph, where edges represents outlives relations, as
a DAG.

## Recursively widening the regions

Say we want to infer the span of some node `'a`. Assume `'a` neighbors (outlives)
`'b, 'c, 'd...`

Since we know the bound, we can say `'a = 'b + 'c + 'd + ...`, since this is the
smallest 'a subject to the outlives conditions.

Now, recursively do the same with `'b, 'c, 'd, ...` Since the graph is acyclic,
this will terminate at some point.

On an implementation note: you can optimize this process by 1. deduplicating
the regions, 2. collapsing sums containing `'static` into `'static`, 3. caching the
nodes to avoid redundant calculations.

## Going further: liveness

Now that we have a closed form for inferring lifetimes, we can do lots of cool stuff.

Liveness of a value is the span starting where the value is declared and ending
where the last access to it is made. This is in contrary to the classical
lexical approach, where the initial lifetimes are assigned as the scopes of the
variables.

Let's start by defining `empty(x)` as the region spanning from x to x (that is,
an empty region at x). Assign every value declared at x a region, `empty(x)`.

Whenever a value of lifetime `'x` is used at some point y, we add a bound `'x:
empty(y)`.

So we essentially expand the region whenever used, effectively yielding the
liveness of the value.

## A happy ending

That's it... The algorithm is really that simple. In fact, you can implement it
in only a 100-200 lines.

## Questions and errata

Ping me at #rust in Mozilla IRC.

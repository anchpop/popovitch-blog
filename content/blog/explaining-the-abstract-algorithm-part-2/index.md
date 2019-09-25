---
title: Explaining the Abstract Algorithm - Part 2
date: "2019-09-08T22:12:03.284Z"
descriptions: Leaving lambda calculus behind
---

Warnings: Haskell knowledge required.

# Sharing nodes

In [Part 1](/explaining-the-abstract-algorithm/) I explained the concept of a *virtual redex*. These happen to be quite important to the Abstract Algorithm. Here's a quote from the introduction of [The Optimal Implementation of Functional Programming Languages](https://www.amazon.com/Implementation-Functional-Programming-Languages-Theoretical/dp/0521621127):

> The formal notion of sharing we shall deal with was formalized in the
seventies by Levy in terms of "families" of redexes with a same origin -
more technically, in terms of sets of redexes that are residuals of a unique
(virtual) redex.

It's not important that you understand all that just yet.

```haskell
id x = x
pair p = (p, p)
hair h = pair (h id)
to_reduce = pair hair
``` 


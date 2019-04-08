---
title: Explaining the Abstract Algorithm
date: "2019-04-08T22:12:03.284Z"
descriptions: Explaining how the abstract algorithm actually works
---

Warnings: Haskell knowledge required.

## Creating a Pathological Example

[In my last post on the Abstract Algorithm](/more-thoughts-on-abstract-algo/), I explained several important terms such as *redex* and *Optimal*. REad that before this one. Now I'm going to explain how the Abstract Algorithm works. Let's set up an example where the Abstract Algorithm really shines.

```haskell
id x = x
pair p = (p, p)
part_1 x = x id
part_2 y = pair (y 3)
to_reduce = part_1 part_2
-- to_reduce == (3, 3)
```

This might seem really convoluted - let's add some type annotations. We'll say that the `3` is an `Int` for simplicity.

```haskell
id x = x
pair p = (p, p)

part_1 :: ((p -> p) -> t) -> t
part_1 x = x id

part_2 :: (Int -> b) -> (b, b)
part_2 y = pair (y 3)

to_reduce :: (Int, Int)
to_reduce = part_1 part_2
-- to_reduce == (3, 3)
```

Now, let's lazily reduce `to_reduce`. First we replace `part_1` with its definition.

```haskell
to_reduce = (\x -> x id) part_2
```

Ok, cool, we can just apply the function.

```haskell
to_reduce = part_2 id
```

Now the only way to reduce this further is to replace `part_2` with its definition.

```haskell
to_reduce = (\y -> pair (y 3)) id
```

Then we can just apply the function.

```haskell
to_reduce = pair (id 3)
```

Now replace `pair` with its definiton.

```haskell
to_reduce = (\p -> (p, p)) (id 3)
```

Then apply the function.

```haskell
to_reduce = (id 3, id 3) 
```

And then I think it's pretty obvious from here how this turns into `(3, 3)`. 

So, there's the same problem as last time. Namely, the `id 3` function gets duplicated and evaluated twice. GHC is smart enough to catch this, even with optimizations turned off, and *share* the value. This is because GHC's evaluation strategy is more complicated than just straightforward lazy evaluation. See [this StackOverflow question](https://stackoverflow.com/questions/3951012/when-is-memoization-automatic-in-ghc-haskell) for more info and why Haskell's strategy does still have issues. 

## What Is a Virtual Redex

If you remember, a `redex` is any expression that can be reduced, such as `(\x -> x) 3` can be reduced to just `3`.

Let's look at the definition of `part_2`:

```haskell
part_2 y = pair (y 3)
```

What you should notice here is that `(y 3)` is *not* a redex, since we don't know what function `y` will be. Just looking inside the `part_2` function, we have no information that would allow us to reduce `part_2`. But, later in the reduction of these functions, we will eventually replace `y` with `id`, which will allow us to easily reduce it. But as of right now, `(y 3)` is not a redex since it can't be reduced. Since we hope that we will one day be able to reduce `(y 3)`, we'll call it a "virtual redex". 

So, is it possible to reduce a virtual redex? It is, and here's how. Here is our example again:

```haskell
id x = x
pair p = (p, p)
part_1 x = x id
part_2 y = pair (y 3)
to_reduce = part_1 part_2
```

We're trying to reduce `to_reduce`, but we want to reduce the `(y 3)` expression within `part_2` early for the sake of demonstration. We identify that we can't do anything with `(y 3)` because we don't know what `y` represents, and the reason we don't know what `y` represents is because it's passed as a parameter to `part_2`. So now we "rise up" and replace `part_2` with it's value.

```haskell
id x = x
pair p = (p, p)
part_1 x = x id
-- part_2 y = pair (y 3)
to_reduce = part_1 (\y -> pair (y 3))
```

Now, we still don't know the value of `y`, so we rise up again:

```haskell
id x = x
pair p = (p, p)
-- part_1 x = x id
-- part_2 y = pair (y 3)
to_reduce = (\x -> x id) (\y -> pair (y 3))
```

Now we can reduce this:

```haskell
id x = x
pair p = (p, p)
-- part_1 x = x id
-- part_2 y = pair (y 3)
to_reduce = (\y -> pair (y 3)) id
```

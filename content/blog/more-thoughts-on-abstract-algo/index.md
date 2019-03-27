---
title: Beginning Thoughts on The Abstract Algorithm
date: "2019-03-28T22:12:03.284Z"
---


The abstract algorithm is an attempt to avoid computing the same data multiple times. Imagine you have the following Haskell function:

```haskell
foo x y = x + x 
``` 

Clearly, not a very useful function. It just doubles `x`, and completely ignores `y`. Now, let's say we pass it the following values.

```haskell 
i = foo (3*3*3) (4^10)
```

There are two obvious ways to actually *call* `foo`. The first is by replacing the arguments with the parameters:

```haskell 
foo x y = x + x 
i = foo (3*3*3) (4^10)
-- becomes...
i = (3*3*3) + (3*3*3)
-- becomes...
i = 54
```

This is clearly not optimal - we will end up doing the `3*3*3` calculation twice. Instead, let's try evaluating before we replace the arguments with the parameters.

```haskell 
foo x y = x + x 
i = foo (3*3*3) (4^10)
-- becomes...
i = foo 27 1048576
-- becomes...
i = 27 + 27
-- becomes...
i = 54
```

This is no better! Since we now evaluate before we substitute, we end up evaluating `4^10` when we didn't have to. 

Haskell solves this issue the first way, with what they call *lazy evaluation*, or *non-strict evaluation*, depending on the nerdiness of the Haskeller you're talking to. Most (all?) imperative languages solve this the second way, called *eager* or *strict evaluation*.


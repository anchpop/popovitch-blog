---
title: Explaining the Abstract Algorithm - Part 1
date: "2019-09-08T22:12:03.284Z"
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

And then we can reduce this one more time.

```haskell
id x = x
pair p = (p, p)
-- part_1 x = x id
-- part_2 y = pair (y 3)
to_reduce = pair (id 3)
```

And now, finally, we can reduce the `(y 3)` since we now know it's really `(id 3)`

```haskell
id x = x
pair p = (p, p)
-- part_1 x = x id
-- part_2 y = pair (y 3)
to_reduce = pair 3
```

It might thing that we're just reducing until we get a value to pass to the virtual redex. But we're actually being a bit more direct than that. Visualize it like a tree:

```haskell
           to_reduce
               $
             /   \  
            /     \  
           /       \  
      part_1 x    part_2 y  ←------------------------------------------------------.
        $            $                                                              \  
      /   \         / \                                                              \   
     /     \       /   \                                                              \  
    /      |       |    \                                                              \ 
   x     id x   pair p   $    ←  our virtual redex                                     /
           |            / \          we want to reduce this but we can't              /
           x           y   3         because we get the value of y from up here - ---`
```

The `$` represents function application, same as normal Haskell. When I write `part_1 x`, it represents the function `part_1` which takes the parameter `x` (I really should write `\x -> part_1 x`, but we have space constraints here). With this visualization we can present a more principled way of reducing the virtual redex `y 3`. Also I've left off the definition of `pair` just for brevity's sake.

So, to evaluate the virtual redex, we start at the `$`.

```haskell
           to_reduce
               $
             /   \  
            /     \  
           /       \  
      part_1 x     part_2 y  
        $            $ 
      /   \         / \ 
     /     \       /   \    
    /      |       |    \   
   x     id x   pair p   $   ← current position
           |            / \ 
           x           y   3    
```

Now we examine the left side of `$` (`y`), and find we don't know what it is! 

```haskell
           to_reduce
               $
             /   \  
            /     \  
           /       \  
      part_1 x     part_2 y  
        $            $ 
      /   \         / \ 
     /     \       /   \    
    /      |       |    \   
   x     id x   pair p   $   
           |            / \ 
           x           y   3    
                       ⮤ current position
```

This means that this expression can't be reduced until we get more information about `y`. If we knew what `y` was going to be, we could reduce this further. To figure out what `y` is, let's jump to where it is introduced.


```haskell
           to_reduce
               $
             /   \  
            /     \  
           /       \  
      part_1 x     part_2 y   ← current position
        $            $    
      /   \         / \
     /     \       /   \    
    /      |       |    \   
   x     id x   pair p   $   
           |            / \ 
           x           y   3    
```

Yes, `y` is introduced at `part_2`. As of right now, we still don't know anything about `y`, so let's defer this question right now. For now, we want to answer if `part_2` is the left part of a function application, because if it were we could find the value of `y` just by looking at the right part of that function application. So let's say we "push the question of 'what is `y`' onto the stack and we arenow  trying to see if the `part_2` function is applied to any values". Let's make a new area to keep track of what mysteries we're solving.

```haskell
                                    ----------------------------
           to_reduce               |     mysteries to solve     |
               $                   | -------------------------- |
             /   \                 | what is part_2 applied to? |  ← current mystery
            /     \                |      what is y?            |
           /       \               ---------------------------- 
      part_1 x     part_2 y   ← current position       
        $            $    
      /   \         / \
     /     \       /   \    
    /      |       |    \   
   x     id x   pair p   $   
           |            / \ 
           x           y   3    
```

You can think of the mysteries as a stack - we're always trying to find the top one, because we need to to solve any below it. If we knew what `part_2` was applied to, whatever it was applied to would be the value of `y`. So, let's "rise up" and see where `part_2` ends up getting used.

```haskell
           to_reduce ← current pos  ----------------------------
               $                   |     mysteries to solve     |
             /   \                 | -------------------------- |
            /     \                | what is part_2 applied to? |  ← current mystery
           /       \               |      what is y?            |
      part_1 x     part_2 y         ---------------------------- 
        $            $    
      /   \         / \  
     /     \       /   \     
    /      |       |    \   
   x     id x   pair p   $   
           |            / \ 
           x           y   3    
```

Now we see `part_2` is being passed as a parameter to `part_1`. This doesn't really help us yet, we're looking for a situation where `part_2` has a parameter passed to it (we want `part_2` to be on the left side of the function application, here it's on the right side). The procedure now is to go down and see how it's used.

```haskell
                     to_reduce                ----------------------------
                         $                   |     mysteries to solve     |
                       /   \                 | -------------------------- |
                      /     \                | what is part_2 applied to? |  ← current mystery
                     /       \               |      what is y?            |
current pos ⭢  part_1 x     part_2 y         ---------------------------- 
                    $            $    
                  /   \         / \  
                 /     \       /   \     
                /      |       |    \   
               x     id x   pair p   $   
                       |            / \ 
                       x           y   3    
```

`part_1` is comprised of a function application. The parameter `x` of `part_1` is `part_2`. Let's go down to see what's on the left side.

```haskell
                     to_reduce                ----------------------------
                         $                   |     mysteries to solve     |
                       /   \                 | -------------------------- |
                      /     \                | what is part_2 applied to? |  ← current mystery
                     /       \               |      what is y?            |
                  part_1 x     part_2 y      ---------------------------- 
                    $            $    
                  /   \         / \  
                 /     \       /   \     
                /      |       |    \   
               x     id x   pair p   $   
                       |            / \ 
                       x           y   3    
```

Finally! `x`, which represents `part_2`, is on the left side of a function appplication! Now let's look at what's on the right side.


```haskell
                     to_reduce                ----------------------------
                         $                   |     mysteries to solve     |
                       /   \                 | -------------------------- |
                      /     \                | what is part_2 applied to? |  ← current mystery
                     /       \               |      what is y?            |
                  part_1 x     part_2 y      ---------------------------- 
                    $            $    
                  /   \         / \  
                 /     \       /   \     
                /      |       |    \   
      current pos ⭢  id x   pair p   $   
                       |            / \ 
                       x           y   3   
```

looks like the `id` function is on the right side - this solves our first mystery. `part_2` is applied to `id`. Let's adjust what we've written down.

```haskell
                     to_reduce                ----------------------------
                         $                   |     mysteries to solve     |
                       /   \                 | -------------------------- |
                      /     \                |         what is y?         |  ← current mystery
                     /       \                ---------------------------- 
                  part_1 x     part_2 y     
                    $            $    
                  /   \         / \  
                 /     \       /   \     
                /      |       |    \   
      current pos ⭢  id x   pair p   $   
                       |            / \ 
                       x           y   3     
```

We now finally know what `y` is, it's the `id` function. With this, we can reduce our `to_reduce` function. Remember, `to_reduce` involved applying `part_1` to `part_2`. So the first step is to apply `part_1` to `part_2`.

```haskell
           to_reduce                         to_reduce       
               $                                 $           
             /   \                             /   \         
            /     \                           /     \        
           /       \                         /       \       
      part_1 x    part_2 y                part_2 y     id x  
        $            $           ->         $            |   
      /   \         / \                   /   \          x   
     /     \       /   \                 /     \             
    /      |       |    \               /      |             
   x     id x   pair p   $            pair     $             
           |            / \                   / \            
           x           y   3                 y   3           
```

In haskell, this would look like this:

```haskell
id x = x                      
pair p = (p, p)               id x = x                             id x = x               
part_1 x = x id           ->  pair p = (p, p)                  ->  pair p = (p, p)        
part_2 y = pair (y 3)         part_2 y = pair (y 3)                part_2 y = pair (y 3)  
to_reduce = part_1 part_2     to_reduce = (\x -> x id) part_2      to_reduce = part_2 id  
                                -- replace part_1 with its           -- reduce
                                -- definition. 
```

(I added the extra step of replacing `part_1` with its definition that we don't have to do when we visualize it as a tree)

Then the next step is to reduce `part_2`!

```haskell
       to_reduce                     to_reduce  
           $                             $      
         /   \                         /   \    
        /     \                       /     \   
       /       \                     /       \  
    part_2 y   id x               pair p      $ 
      $          |         ->                / \
    /   \        x                        id x  3
   /     \                                  |      
  /      |                                  x   
pair     $                                      
        / \                                     
       y   3                                    

id x = x                   
pair p = (p, p)        ->  id x = x                               id x = x
part_2 y = pair (y 3)      pair p = (p, p)                    ->  pair p = (p, p)
to_reduce = part_2 id      to_reduce = (\y -> pair (y 3)) id      to_reduce = pair (id 3)
                             -- replace part_2 with its
                             -- definition. 
```

Then we just apply `id`!

```haskell
   to_reduce             to_reduce   
       $                     $       
     /   \                 /   \     
    /     \      ->       /     \    
   /       \             /       \   
pair p      $         pair p      3  
           / \                         
        id x  3                      
          |                          
          x                          


id x = x                     
pair p = (p, p)          ->  pair p = (p, p)                   ->  pair p = (p, p)               
to_reduce = pair (id 3)      to_reduce = pair ((\x -> x) 3)        to_reduce = pair 3
                               -- replace id with its
                               -- definition. 
```

We have reduced the "virtual redex"! `(y 3)` has been converted to its true self, `3`. But the interesting thing is that we've reduced the virtual redex in an *optimal number* of steps. Let's be more precice. Remember how we were moving around that *current position* arrow all around the graph earlier? Let's label all the edges that we visited on the tree.

```haskell
           to_reduce            
               $                
             /   \              
           c/     \b            
           /       \            
      part_1 x    part_2 y      
        $            $          
      /   \         / \         
    d/     \e      /   \        
    /      |       |    \       
   x     id x  pair p    $      
           |           a/ \     
           x           y   3    
```

We started at `(y 3)`, so our initial position is the `$` connected to the edge labelled `a`. You can think of a reduction as combining multiple edges.

```haskell
           to_reduce                         to_reduce                     to_reduce     
               $                                 $                             $         
             /   \                             /   \                         /   \       
           c/     \b                       bcd/     \e                      /     \      
           /       \                         /       \                     /       \     
      part_1 x    part_2 y                part_2 y     id x             pair p      $    
        $            $           ->         $            |       ->           abcde/ \   
      /   \         / \                   /   \          x                      id x  3  
    d/     \e      /   \                 /     \                                  |      
    /      |       |    \               /      |                                  x      
   x     id x   pair p   $            pair     $                                         
           |           a/ \                  a/ \                                        
           x           y   3                 y   3                                       
```

First we reduce the application of `part_1` to `part_2`, which has the effect of combining the `b`, `c`, and `d` edges, which we'll call `bcd`. Then we reduce the application of `part_2` to `id`, which combines the `a`, `bcd`, and `e` edges. This is the absolute minimum we have to do to convert the virtual redex `(y 3)` to a the redex `(id 3)`.

What we're really done here is described in a very high level way how to 1) find the shortest path between a mysterious element in a virtual redex and 2) use that path to optimally reduce a virtual redex. So, any reduction algorithm that is optimal must:

    1) Compute the normal form of am expression without reducing any path corresponding to a virtual redex more than once.

    2) Compute the normal form of am expression without reducing any uneccsary path.

This concept of virtual redexes is very important for understanding the abstract algorithm. For more information, see part 2 once it comes out!
---
title: Why Do We Need Another Programming Language 
date: "2019-03-28T23:46:37.121Z"
---

The languages that we have available to us for writing games today are inadequate. I won't attempt to prove that here - I don't expect to change the mind of anyone who doesn't already agree with me. You want a few things in any language for writing games:

1) Beginner friendliness

2) Good tooling

3) Efficiency

4) Enabling fast prototyping

5) Steering users towards writing bug-free code, and preventing bugs statically wherever possible

6) Self-embedding

7) Cross platform

You don't have to have all of these, but each one you lack is a room for improvement. So, how do the mainstream programming languages stack up? C++ isn't beginner friendly and the community hasn't standardized around any particular build system, not to mention writing a large bug-free program in C++ is harder than necessary. Rust is difficult to learn and build times can make it difficult to prototype. Python is easy to learn but inefficient. Haskell is both difficult to learn and experiences pauses from garbage collection, and has abysmal tooling. C# experiences pauses from garbage collection and does not do a sufficiently good job at steering users towards writing high-quality code. And of course, none support self-embedding out of the box.

Games can be written in these languages, and it can be a fun, rewarding, and profitable experience. However, all of my experience with writing games has pointed me toward the conclusion that we need a better language capable of encoding more powerful abstractions. And such a language should have compilation and build systems present as *first-class standard library tools*, because we want to write our game engine editor in the same language as our games, and support user mods, etc.

Thus I present Zoda, my next project. The Zoda compiler and build system will be written in Rust, and it will not be bootstrapped. All Zoda programs will function identically on all platforms except in cases where you need to use a feature not supported on a certain platform. 

Zoda programs will be inherently sandboxed and the build process will have no side effects outside of caching, so running untrusted code will be as simple as not providing it with the permissions you don't want it to have. This makes depending on 3rd party code a no-brainer even for those of us who are security-minded.

People tend to use more libraries and 3rd party code in Javascript than Haskell. Is this a virtue? I say yes. You should write as little code as humanly possible, and reuse as much external code as humanly possible, including for very simple cases such as left-pad. How can we encourage this behavior of not reinventing the wheel? I suggest:

1) A build system that makes it trivial to pull in external dependencies 

2) Design the language to encourage smaller, more modular packages

3) Design the tooling around the languages to make it easy to write documentation for your project. The language and tooling should step in to encourage library authors to write good documentation *at every possible opportunity*.

4) Have a small standard library. The only things that should be included in the standard library are things which cannot be reasonably divorced from it, such as magic functions that cannot be written otherwise or data structures for which the language provides some syntax sugar.

5) Have very aggressive DCE, to keep binaries small if you only use a small part of a big library.


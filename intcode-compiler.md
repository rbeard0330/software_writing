---
layout: default
title: "Building an Intcode Compiler in Rust"
permalink: /intcode-compiler/
---
## What am I even reading?
This is the index page for my ongoing project to build a compiler targeting the Intcode language that was created for the 2019 Advent of Code programming challenge.[^1]

[^1]: The Intcode language first appears on [Day 2](https://adventofcode.com/2019/day/2).  It's extended on [Day 5](https://adventofcode.com/2019/day/2), [Day 7](https://adventofcode.com/2019/day/7), and [Day 9](https://adventofcode.com/2019/day/9), which contains the full specification for the language.  Later challenges provide Intcode problems as inputs.

## Who are you?
Among other things, I'm a recovering lawyer and the solo founder of Sapphire Legal and Tax Software, a legal-tech startup that builds intuitive and capable web applications for lawyers and tax professionals.  You can read more about me and Sapphire here.

It may be more interesting to explain who I'm not:
* I don't have a CS degree, and I don't have any experience working as an engineer at a big tech company.  My programming experience is entirely learn-by-doing.
* I don't know what I'm doing, and I don't have any particular reason to believe I can successfully write a compiler.  That means that this series will probably be low on elegant insights, but maybe it will provide a better experience of the beginner's journey into this subject.

## Why are you doing this?
One inspiration for this project was a conversation with a friend about the confusing and difficult nature of concurrent programming, which lead me to the conclusion that "computing is like an onion where after you understand what’s happening at one layer, you then realize that it’s a barely coherent abstraction over a swamp of seething chaos."  This is not a great description of an onion, but it captures an important element of my experience learning to craft software.  If you get far enough learning about a high-level framework like Django, you start to see more and more traces of the elaborate clockwork of Python metaclasses and dynamic descriptors that make the whole thing work.  And if you study Python long enough, more and more CPython implementation details poke through.  Experienced systems programmers in languages like C or Rust are always talking about what the compiler is likely to do with different bits of code that express the same semantics.  Beyond the compiler, there's an immense amount of complexity having to do with caching and branch-prediction and all the other performance tricks that modern CPUs learn.

What I hope to gain from this project is to develop a somewhat more ordered mental model of one of these layers of seething chaos.  What is "the stack" and why does it exist?  What really happens when you call a function?  Why does recursion usually perform poorly, and what do functional-programming types mean when they talk about tail-call optimization?  What is a calling convention or an ABI?  What is a linker?

One way to get a grip on these question is to just try to do it, see what problems I run into, and see if my solutions fall into the same shape as some of these concepts.  That's what I intend to do with this project.

## How is this going to work?
### The problem definition
"Taming seething chaos" is a noble-sounding goal, but it's not obviously implementable.  To make things more concrete, our compiler is going to start with a simplified C-like language and target the Intcode VM.  Initially, our compiler is going to start with a very, very limited source language and produce correct, but low-quality code.  Once we have something that works, we'll start making the language richer and implementing optimizations in the compiler's output.

### Project language
Everything will be written in Rust.  Rust has a lot of really cool high-level features while still producing fast native code.  Plus it does a lot to maintain type and memory safety, which I expect will help us implement our compiler correctly and avoid breaking it as we tinker with it over time.

### Sources
My main reference for this project is *Engineering a Compiler* by Linda Torczon and Keith D. Cooper.  I'll pilfer insights from anywhere else I can find them as we go.  Comments from readers are very, very welcome (email me [here](mailto:robert.a.beard@gmail.com)).

The one thing I'm going to religiously avoid is peeking at other people's solutions.  This project is by no means an original idea, and there are many other implementations of the same concept (often more ambitious and doubtless better executed).  We'll see if I stick to that promise when the going gets tough...

### Code
Everything I write here will be posted on [GitHub](https://github.com/rbeard0330/intcode-compiler), with different stages of the project set up as branches.  The code can be freely used for any purpose.  The blog itself is copyrighted, but please contact me if you want to do something with it that might not be fair use, and I'm sure we can work out a free license for anything noncommercial.


## Index of posts
If we're doing our jobs as engineers well, our understanding of the problem will evolve as we start to design and implement a solution.  That means that any detailed schedule is going to evolve and change as I actually write these posts.  With that caveat, see below for my *best guess* about how this project plays out.  As the posts get written, this index will be updated with links and to reflect any changes in our plans.
1. [Introducing the IVM and designing Low-Level IR (LLIR) v1](/2022-01-31-intcode-compiler-in-rust-1.md)
2. [Building an IVM runtime](/2022-02-10-intcode-compiler-in-rust-2.md)
3. Emitting Intcode from LLIR v1
4. Introducing C-- (our toy source language)
5. Variables and LLIR v2
6. Procedure calls, Part I
7. Procedure calls, Part II and LLIR v3
8. Control flow and designing Graphical IR (GIR) v1
9. Parsing C-- into GIR v1
10. Our first end-to-end (nonoptimizing) compiler
11. Introduction to optimizations
12. \[A few posts on optimization]
13. Extending C-- with arrays
14. Extending C-- with structs
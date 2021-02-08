---
layout: post
title:  "Writing a Domain Specific Language in Rust, Part 1"
date:   2021-02-07 18:00:00 -0800
categories: Rust
---

**tl;dr** - This _Part 1_ is mostly fluffy stuff.  Those wishing to skip the majestic but emotionally draining backstory can skip to part 2!

Before anyone sets their hopes too high, the subject of this article is my first big venture into Rust programming.  Nevertheless, I hope the series adds up to more than a random walk and can help others trying to start out.

## Rust

If you're here, you've probably at least heard of [Rust](ttps://www.rust-lang.org). Other upstarts have tried to eat C/C++'s lunch in one way or another, but Rust brought something dramatic to the table: the fulfilled promise of speed and safety.  The secret sauce is Rust's borrow checker, built above rigorous concepts of memory ownership and lifetime. If your program segfaults, creates data races, uses memory after freeing, or commits related sins, the Rust compiler has a bug for letting that happen. If your code is too complex to analyze, Rust demands an explanation using lifetime annotations or refactoring into a more transparent design.

## The Airing of the Greivances

Rust is not a beginner's language. The Overflow had a [article](https://stackoverflow.blog/2020/01/20/what-is-rust-and-why-is-it-so-popular) discussing a few perspectives depending where you're coming from. My own background is mainly C/C++ and all the curly braces and semicolons lulled me into a false sense of comfort.

Online searches show many developers struggling to satisfy Rust's borrow checker. However, if you come from C heritage, you're forcibly drilled in machine memory concepts. I was surprised by the new errors the checker throws, but C/C++ peeps have a head start in the fundamentals. Even when the borrow problems were non-bugs _today_, seeing potential _future_ bugs was welcome.

The biggest problems in this effort were the false starts. Rust's library ecosystem swarms with activity, but lacks especially in documentation. Approaches that appeared promising at first, e.g. the lalrpop parser, lead me to eventually switch gears in dissatisfaction.

Another initial irritation was Rust's module system.  Rust module handling is entwined with [cargo](https://doc.rust-lang.org/cargo) which is Rust's package manager and build system. Coming from the dirty but obvious #includes of C/C++, I expected my project directory structure to be _the_ module layout.  Alas, cargo (or rustc?) would complain a module right under its nose could not be found.

## Why a domain specific langauge?

As a realistic evaluation of Rust, DSL's are a juicy choice. DSL's have many moving parts and complex error handling requirements. They avoid the need for a GUI and lend themselves well to unit and fuzz testing. After the initial heavy lift, DSL's are fun to hack on.

The DSL I'll talk about is brink. Brink is a curly-brace-semicolon style declarative language for linking files together from constituent parts. The primary use case is defining flash or other non-volatile memory images for embedded systems. Embedded systems often need a boot image that falls somewhere between a dumb blob of bits and whole file system.

Here's a an simple brink source file:

    // Create a file with a single string
    section hello { wrs "Hello World!\n"; }
    output hello;

## Goals

My aim is to elaborate on my experiences implementing compiler basics in Rust and share the library choices that worked.  These recommendations come after screwing up completely a couple times. For example, compilers love graph structures, but Rust frowns on the zoo of memory pointers that tie graphs together in C. Compilers are often built with help form lexer and parser generators, e.g. the venerable lex and yacc. What would Rust offer?

Secondly, I wanted to get all the compiler infrastructure correct before heaping on brink language features. Error reporting, test infrastructure, command line processing should all be in proper final form as early as possible in development. The point is not this particular DSL, but to describe a reusable jumping off point for your own DSL ideas.

Finally, more a hope than a goal, is to get feedback on how this approach could improve. Being new to rust and no particular authority in compiler design, others may have something helpful to say.


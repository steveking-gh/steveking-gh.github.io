---
layout: post
title:  "Writing a Domain Specific Language in Rust, Part 1"
date:   2021-02-07 18:00:00 -0800
categories: Rust
---

**tl;dr** - This _Part 1_ is just fluffy stuff.  Those wishing to skip this majestic but emotionally draining backstory can skip to [Part 2]({% link _posts/2021-02-07-writing_a_dsl_in_rust_part_2.markdown %}).

Before anyone sets their hopes too high, the subject of this article is my first big venture into Rust programming.  Nevertheless, I hope the series can help others trying to start out.

## Rust

If you're here, you've probably at least heard of
[Rust](ttps://www.rust-lang.org). Other upstarts have tried to eat C/C++'s lunch
in one way or another, but Rust brought something dramatic to the table: the
fulfilled promise of speed and safety.

For C/C++ people, ever had a use-after-free bug?  Here's a toy example:

```c++
    #include <string>
    #include <iostream>
    
    using namespace std;
    
    const char* foo() {
        string badmojo("abc");
        return badmojo.c_str();
    }
    
    int main() {
        cout << foo() << endl;
    }
```

The `badmojo.c_str()` function returns a pointer to stack memory that disappears as soon as
foo returns.  That's a lovely use-after-free
[Heisenbug](http://simonb.com/blog/2009/07/10/heisenbug/) in the caller, or the
caller's caller, or who knows where.  GNU g++ with `-Wall -Wextra` compiles
clean, so no help from the compiler.

Trusty [valgrind --track-origins=yes](https://valgrind.org/) catches this bug
as use of an uninitialized value in a standard library function buried in `cout
<<`. Valgrind correctly blames the problem on a stack allocation somewhere, but
doesn't implicate `foo()`.

And hey!  No compiler warning about that `int` return code from main not being
handled?

In Rust, entire categories of bugs like this cause compile time errors.  The
compiler barfs right on the root of your bug, no hunting around.

## Why create a domain specific language?

As a realistic evaluation of Rust, DSL's are a juicy choice. DSL's have many moving parts, pointer laden tree structures and complex error handling requirements.

The DSL I'll talk about is [Brink](https://github.com/steveking-gh/brink). Brink is a curly-brace-semicolon style declarative language for linking files together from constituent parts. The primary use case is defining flash or other non-volatile memory images for embedded systems that don't want the complexity of a full filesystem.

Here's a simple and complete Brink source file:

    // Create a file with a single string
    section hello { wrs "Hello World!\n"; }
    output hello;


Next: [Part 2]({% link _posts/2021-02-07-writing_a_dsl_in_rust_part_2.markdown %}).
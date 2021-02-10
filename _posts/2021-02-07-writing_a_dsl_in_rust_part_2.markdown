---
layout: post
title:  "Writing a Domain Specific Language in Rust, Part 2"
date:   2021-02-07 21:01:00 -0800
categories: Rust
---

**tl;dr** - _Part 2_ is about setup, directory and workspace structure.  Back to [Part 1]({% link _posts/2021-02-07-writing_a_dsl_in_rust_part_1.markdown %}).

## Prereqs

If you don't already have Rust installed, then start with the [official guide](https://www.rust-lang.org/learn/get-started).

### Windows WSL2 dependencies

I use [WSL2](https://docs.microsoft.com/en-us/windows/wsl/install-win10) running Ubuntu 20.04 for most development.  Rust runs in native windows too except for fuzz testing which has some external Linux dependencies.

I like the [cargo-edit](https://crates.io/crates/cargo-edit) extension, which has a few dependencies not installed by default if you go with WSL2.

    sudo apt install openssl-dev
    sudo apt install pkg-config

No you can install [cargo-edit](https://medium.com/r/?url=https%3A%2F%2Fcrates.io%2Fcrates%2Fcargo-edit):

    cargo install cargo-edit

You'll also need the [fuzz testing tool](https://github.com/rust-fuzz/cargo-fuzz).  More about fuzz testing later.

    cargo install cargo-fuzz

### Editor

I use and much appreciate [Visual Studio Code](https://code.visualstudio.com/) for Rust development.  Interop with WSL2 environments is amazingly smooth.  Rust-wise, there are _a lot_ of extensions available, but I'm only using the awesome **rust-analyzer** extension.  You'll also want the **CodeLLDB** extension for debugging.

## Project directory structure

After trial and error, the following structure worked without causing surprises.  There are other ways to organize a large-ish project.

| ![project directory structure diagram](/images/project_structure.svg) | 
|:--:| 
| *Project directory structure* |

Let's examine a few points in detail.

### The root package
The package identified in the `Cargo.toml` of the top directory in the project is the [root package](https://doc.rust-lang.org/cargo/reference/workspaces.html#root-package).  This is __MyProject__ in the diagram above.  List all the other local packages (usually your libraries) in the \[workspace\] part of the Cargo.toml file.  These are the `foo` and `bar` libraries in the diagram.

### Local library dependencies
The root package `Cargo.toml` refers to local dependencies using the `path` property.  For example if the root project depends on the local `foo` library:

    # root Cargo.toml
    [dependencies]
    foo = { path = "./foo" }

Similiarly, if the `foo` library depends on the local `bar` library, then we need to refer up one directory level with `../`

    # foo's Cargo.toml
    [dependencies]
    bar = { path = "../bar" }

## Fuzz Early and Often

[Fuzz testing](https://en.wikipedia.org/wiki/Fuzzing) means blasting your library with pseudo-randomized input to try to cause crashes.  Good fuzz testers try to focus on input conditions that matter rather than pure randomness.  For example, consider a program that takes a 32-bit integer input.  Values like 0, 1, and -1 are more likely to be coding corner cases than the wasteland between 2 and 4294967294.  You don't want a fuzz tester spending too much time in the boring middle.

In the diagram above, each library, e.g. __foo__ and __bar__, can have its own [fuzz testing](https://en.wikipedia.org/wiki/Fuzzing) sub-project.  Fuzz testing your libraries as development progresses finds problems early and encourages real error handling.  Cargo-fuzz will not complain if your library returns errors, but will count all crashes as bugs.  It doesn't matter if you saw the crash coming with a `panic!()`, it's still a crash.  __You'll need to refactor all crashes that result from bad input as errors returned to the caller__.  After that tidying up, fuzz testing will still find crashes, but these are the interesting cases you did _not_ see coming.

For Rust, the easiest fuzzing choice is [cargo-fuzz](https://github.com/rust-fuzz/cargo-fuzz).

    cargo install cargo-fuzz
    cd <my library dir>
    cargo fuzz init

The `cargo fuzz init` command creates a `fuzz` subdirectory containing its own package and `Cargo.toml` file.  Typical directory structure looks like this:

```
    fuzz
    ├── Cargo.toml
    └── fuzz_targets
        └── fuzz_target_1.rs
```

You must edit the fuzz_target_1.rs file to suit your library.  For example, here's the [fuzz wrapper function](https://github.com/steveking-gh/brink/blob/master/ast/fuzz/fuzz_targets/fuzz_target_1.rs) used by Brink's abstract syntax tree (AST):

```rust
#![no_main]
use libfuzzer_sys::fuzz_target;
use ast::Ast;
use std::io::Write;
use diags::Diags;

// The fuzzer calls this function repeatedly
fuzz_target!(|data: &[u8]| {
    if let Ok(str_in) = std::str::from_utf8(data) {
        // Set the verbosity to 0 to avoid console error
        // messages during the test.
        let mut diags = Diags::new("fuzz_target_1",str_in, 0);
        let _ = Ast::new(str_in, &mut diags);
    }
});
```

In the code above, the fuzz tester supplies a randomized `u8` array.  To test AST logic, we pretend this input is the user's Brink source code.  You can ignore the `diags` object for now and otherwise we convert from `[u8]` and supply the "source code" to the AST constructor.

The fuzz packages in each library directory don't do anything during a normal `cargo build`.  At the time of writing (Rust 1.49), you need the +nightly option when running the fuzzer.  The typical commands are:

    cd <library containing a fuzz directory>
    cargo +nightly fuzz run fuzz_target_1

For better results, you'll need to supply a _corpus_ of inputs that give the fuzzer a clue about your inputs.  In the case of a DSL like Brink, the corpus is just a bunch of Brink input files copied from the tests directory.

## Libraries used in Brink

This isn't a complete list, but highlights worth mentioning.

|__Library__| __Purpose__ |
| [clap](https://docs.rs/clap) | command line handling |
| [fern](https://docs.rs/fern) | Debug logging |
| [logos](https://docs.rs/logos) | Tokenizing (aka lexing) of user source files |
| [anyhow](https://docs.rs/anyhow) | Friendly error (Result<>) enhancements |
| [indextree](https://docs.rs/indextree) | Rust friendly tree structures |
| [codespan-reporting](https://docs.rs/codespan-reporting) | Beautiful reporting of user source code errors |
| [parseint](https://docs.rs/parse_int) | Nice string to integer conversion |


What about a [parsing library](https://lib.rs/parsing)?  The tl;dr is that Brink uses a hand written recursive descent parser with Pratt parsing for expressions.  More on this later.

At the start of the project, I wanted (still do!) the following:
* To describe Brink's language using [BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)
* A parser generator that produced real and visible Rust code.  Hiding the complexity of a parser generator behind macros sounds terrifying.  The additional build step is a small price to pay.

That led me to [lalrpop](https://docs.rs/lalrpop) written by none other than core Rust developer [Niko Matsakis](https://github.com/nikomatsakis).  Lalrpop even had built-in lexing with ability to handle C style comments.  The documentation wasn't extensive, but just enough and I gave lalrpop a serious try.  Eventually though, I couldn't generate the kind of user error messages I wanted and moved on.
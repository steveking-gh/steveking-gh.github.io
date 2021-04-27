---
layout: post
title:  "Writing a Domain Specific Language in Rust, Part 2 - Layout and Libraries"
date:   2021-02-07 21:01:00 -0800
categories: Rust
---

**tl;dr** - _Part 2_ is about setup, directory and workspace structure.  Back to [Part 1]({% link _posts/2021-02-07-writing_a_dsl_in_rust_part_1.markdown %}) or ahead to [Part 3]({% link _posts/2021-04-25-writing_a_dsl_in_rust_part_3.markdown %}).

![House Foundation](/images/house_foundation.jpg)

## Prereqs

If you don't already have Rust installed, then start with the [Install Rust guide](https://www.rust-lang.org/tools/install).  Like the guide recommends, let `rustup` take control rather than your OS's package manager (dnf, apt, etc).

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

I use and much appreciate [Visual Studio Code](https://code.visualstudio.com/) for Rust development.  Interop with WSL2 environments is amazingly smooth.  Rust-wise, there are _a lot_ of extensions available, but I'm using the awesome **rust-analyzer** extension.  You'll also want the **CodeLLDB** extension for debugging.

## Project directory structure

After much trial and error, the following structure worked without causing surprises.  There are other ways to organize a large-ish project.

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

Similarly, if the `foo` library depends on the local `bar` library, then we need to refer up one directory level with `../`

    # foo's Cargo.toml
    [dependencies]
    bar = { path = "../bar" }

### Issue #2224
Modularizing a big project into private libraries is good practice in general and in Rust.  Sadly, the `cargo publish` command expects a flat earth where an app's internal libraries must exist as free-standing crates!  See [Closed and unfixed Issue #2224](https://github.com/rust-lang/rfcs/pull/2224).  In practical terms, you must choose:
* pollute [crates.io](https://crates.io/) with internal-only libraries
* create a monolithic code structure without the use of internal libraries
* Don't use `cargo publish` to publish your app

I fell ass-backward into the third option.

If you decide not to use internal modules in your app, you'll have a harder time with fuzz testing as described in [Part 3]({% link _posts/2021-04-25-writing_a_dsl_in_rust_part_3.markdown %}).  __IMHO, fuzz testing provides far more value to DSL development than `cargo publish`.__ 

![Library](/images/library.jpg)
## Libraries used in Brink

This isn't a complete list, but highlights worth mentioning.

|__Library__| __Purpose__ |
|-----------|-------------|
| [clap](https://docs.rs/clap) | command line handling |
| [fern](https://docs.rs/fern) | Debug logging |
| [logos](https://docs.rs/logos) | Tokenizing (aka lexing) of user source files |
| [anyhow](https://docs.rs/anyhow) | Friendly error (Result<>) enhancements |
| [indextree](https://docs.rs/indextree) | Rust friendly tree structures |
| [codespan-reporting](https://docs.rs/codespan-reporting) | Beautiful reporting of user source code errors |
| [parseint](https://docs.rs/parse_int) | Nice string to integer conversion |


What about a [parsing library](https://lib.rs/parsing)?  I wanted to use one, but the end result of my travails is that Brink uses a hand written recursive descent parser with precedence climbing.  More on this later.

At the start of the project, I wanted (still do!) the following:
* To describe Brink's language using [BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)
* A parser generator that produced real and visible Rust code.  Hiding the complexity of a parser generator behind macros is too opaque for my taste.  The additional build step is a small price to pay for transparency of operation.
* A parser that produced an abstract syntax tree, or made it straightforward for me to do so.

That led me to abortive attempt to use [lalrpop](https://docs.rs/lalrpop) written by none other than core Rust developer [Niko Matsakis](https://github.com/nikomatsakis).  Lalrpop even had built-in lexing with ability to handle C-style comments.  The documentation wasn't extensive, but just enough and I gave lalrpop a serious try.  However, I couldn't generate the error messages I wanted.  Eventually I switched to hand written recursive descent.

Next: [Part 3]({% link _posts/2021-04-25-writing_a_dsl_in_rust_part_3.markdown %}).

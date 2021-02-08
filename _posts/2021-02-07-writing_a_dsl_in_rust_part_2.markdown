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

Notice that each library can have its own [fuzz testing](https://en.wikipedia.org/wiki/Fuzzing) sub-project.  Fuzz testing your libraries as development progresses finds problems early and encourages real error handling.  Cargo-fuzz will not complain if your library returns errors, but will count all crashes as bugs.  It doesn't matter if you saw the crash coming with a `panic!()`, it's still a crash.  __You'll need to refactor all crashes as errors returned to the caller__.  After that tidying up, fuzz testing will find crashes you did _not_ see coming.

For Rust, the easiest fuzzing choice is [cargo-fuzz](https://github.com/rust-fuzz/cargo-fuzz).

    cargo install cargo-fuzz
    cd <my library dir>
    cargo fuzz init

The `cargo fuzz init` command creates a `fuzz` subdirectory containing its own package and `Cargo.toml` file.  Typical directory structure looks like this:

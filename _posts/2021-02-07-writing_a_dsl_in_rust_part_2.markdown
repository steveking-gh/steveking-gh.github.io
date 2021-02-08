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

I use [WSL2](https://docs.microsoft.com/en-us/windows/wsl/install-win10) running Ubuntu 20.04 for most development.  That's just personal preference and Rust runs fine in native windows.

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

Organizing libraries in a way that other libraries can find them was a headache.  After trial and error, the following structure works without causing surprises:

| ![project directory structure diagram](/images/project_structure.svg) | 
|:--:| 
| *Project directory structure* |

Let's examine a few points in detail.

### The root package
The package identified in the top Cargo.toml of the project is the [root package](https://doc.rust-lang.org/cargo/reference/workspaces.html#root-package).  This is __MyProject__ in the diagram above.  List all the other local packages (usually your libraries) in the \[workspace\] part of the Cargo.toml file.  These are the __foo__ and __bar__ libraries in the diagram.

### Local library dependencies
The root package Cargo.toml refers to local dependencies using the _path_ property.  For example if the root project depends on the local foo library:

    # root Cargo.toml
    [dependencies]
    foo = { path = "./foo" }

Similiarly, if the foo library depends on the local bar library:

    # foo's Cargo.toml
    [dependencies]
    bar = { path = "../bar" }




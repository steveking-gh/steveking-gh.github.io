---
layout: post
title:  "Writing a Domain Specific Language in Rust, Part 3 - Fuzz Early and Often"
date:   2021-02-07 18:00:00 -0800
categories: Rust
---

**tl;dr** - _Part 3_ is fuzz testing your internal modules.  Back to [Part 2]({% link _posts/2021-02-07-writing_a_dsl_in_rust_part_2.markdown %}).

![Fuzzy Dice](/images/fuzz_dice.jpg)

## Fuzz Testing

[Fuzz testing](https://en.wikipedia.org/wiki/Fuzzing) means blasting your creation with pseudo-randomized input to try to cause crashes.  Good fuzz testers focus on input conditions that matter rather than pure randomness.  For example, consider a program that takes a 32-bit integer input.  Values like 0, 1, and -1 are more likely to be coding corner cases than the wasteland between 2 and 4294967294.  You don't want a fuzz tester spending too much time in the boring middle.

In the diagram above, each library, e.g. __foo__ and __bar__, can have its own [fuzz testing](https://en.wikipedia.org/wiki/Fuzzing) sub-project.  Fuzz testing your libraries as development progresses finds problems early and encourages real error handling.  Cargo-fuzz will not complain if your library returns errors, but counts all crashes as bugs.  It doesn't matter if you saw the crash coming with a `panic!()`, it's still a crash.  __You'll need to refactor all crashes that result from bad input as errors returned to the caller__.  After that tidying up, fuzz testing will still find crashes, but these are the interesting cases you did _not_ see coming.

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

The fuzz packages in each library directory don't do anything during a normal `cargo build`.

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

### Nightly

At the time of writing (Rust 1.51), you need the +nightly option when running the fuzzer.  The typical commands are:

    cd <library containing a fuzz directory>
    cargo +nightly fuzz run fuzz_target_1 -- -max_len=256 -timeout=5

The `-max_len` option sets the maximum size of the generated input files.  This
option avoids wasted test time on oversize whitespace excursions and so on.
In the case of Brink, a fuzz tester can wreak plenty of havoc in a 256 byte source file.


![Young Frankenstein](/images/young_frankenstein-980x520.jpg)

## The Corpus

For more efficient bug finding, you can supply the fuzzer with example inputs.  The term of art for this feed material is a _corpus_.  In the case of a domain specific language, the corpus is a bunch of sample programs.  For Brink, I usually just copy my unit tests into the corpus directory.  For example:

    $ mkdir -p ast/fuzz/corpus/fuzz_target_1
    $ cp tests/*.brink ast/fuzz/corpus/fuzz_target_1/
    $ cd ast
    $ cargo +nightly fuzz run fuzz_target_1 -- -max_len=256 -timeout=5

On my system, The fuzz tester executes about 47K tests per second.

## The Fuzzer Found a Crash!

When the fuzz tester finds a crash, you'll see the path the responsible source file.  Helpfully, the fuzzer provides the exact command line to reduce the failing input to a minimum example.  I generally always run this extra step and then do further minimization by hand.

__DO NOT JUST FIX YOUR BUG!__  Instead, copy the failing input to your tests directory as a corresponding integration test!  At the time of writing, Brink has 18 such cases, starting from fuzz_found_1.brink on up.  For example, here's `fuzz_found_15.brink`:

    section foo {
        print 1,;
    }
    
    output foo;

Well, I never thought to check that trailing ',' before the semicolon.  Fortunately, the fuzzer brute forces its way into these untidy corners.

Here's the corresponding `tests/integration.rs` test for `fuzz_found_15.brink`.  We expect this test to fail since the input breaks Brink's grammar rules, but we want a nice error message instead of a `panic!()`.

    #[test]
    fn fuzz_found_15() {
        let _cmd = Command::cargo_bin("brink")
        .unwrap()
        .arg("tests/fuzz_found_15.brink")
        .assert()
        .failure()
        .stderr(predicates::str::contains("[AST_21]"));
    }

After this due diligence, then go a fix your actual bug.
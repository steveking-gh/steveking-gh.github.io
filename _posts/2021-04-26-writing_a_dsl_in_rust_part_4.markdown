---
layout: post
title:  "Writing a Domain Specific Language in Rust, Part 4 - Program Flow"
date:   2021-04-26 10:00:00 -0800
categories: Rust
---

tl;dr - Part 4 is about overall compiler flow. Back to [Part 3]({% link _posts/2021-04-25-writing_a_dsl_in_rust_part_3.markdown %})



![Portland Arial View](/images/dan-meyers-kzSNNqqS3Qs-unsplash_small.jpg)
# Program Flow
Let's take a simplified top-down look at the processing flow used by [Brink](https://github.com/steveking-gh/brink).
---


![Read source](/images/brink_flow_step1.svg)

Assuming your DSL doesn't take enormous source files, just do the boring simple thing and read the whole source into a string at once.  Even with full featured error handling and line feed conversion, a single long statement can do the job.  Here's Brink's example from [main.rs](https://github.com/steveking-gh/brink/blob/master/src/main.rs):

```rust
    let str_in = fs::read_to_string(&in_file_name)
        .with_context(|| format!(
                "Failed to read from file {}.\nWorking directory is {}",
                in_file_name, env::current_dir().unwrap().display()))?
        .replace("\r\n","\n");
```
---

![Tokenize](/images/brink_flow_step2.svg)

Tokenize, aka _lex_, your string into enum values meaningful in your DSL.  Brink uses the wonderful [Logos](https://github.com/maciejhirsz/logos) tokenizer.  Logos takes an enum of macro-fied regular expressions and otherwise does all the work for you.

By necessity, some regular expressions get complicated.  For example, Brink skips C style line and block comments during tokenizing using the following Logos enum entries.  You can find all the tokens in [ast.rs](https://github.com/steveking-gh/brink/blob/master/ast/ast.rs)

```rust
    // Comments and whitespace are stripped from user input during processing.
    // This stripping happens *after* we record all the line/offset info
    // with codespan for error reporting.
    #[regex(r#"/\*([^*]|\*[^/])+\*/"#, logos::skip)] // block comments
    #[regex(r#"//[^\r\n]*(\r\n|\n)?"#, logos::skip)] // line comments
```

## Don't Attach Data to Your Token Enum Elements
Rust allows enum elements to carry one or more data objects.  There is nothing wrong with Logos in this regard, but I recommend _not_ using this facet of rust for tokens.  The problem is that you'll need to match your tokens in a variety of circumstances and the extra complexity of the unstructured tuple gets in the way.  Instead, create a first class struct to contain the enum value and whatever else you need and put that struct into your token vector.  In the case of brink, per-token information boiled down to three things:
* The token enum value
* The source code location of the token
* The string value of the token trimmed of surrounding whitespace

---

![AST](/images/brink_flow_step3.svg)

Now things get juicy.  We can handle steps 1 and 2 in couple screenfulls of code.  Converting the token stream into a syntax tree requires a lot more effort.

An obvious approach is to use a parser generator such as [lalrpop](https://docs.rs/lalrpop).  As mentioned in [Part 2]({% link _posts/2021-02-07-writing_a_dsl_in_rust_part_2.markdown %}), there are pros and cons.  After several false starts I eventually rolled my own recursive decent parser.  Most large compilers use their own recursive decent parsing as well.  Brink is definitely not large, but more parsing discussion can come in a later post.

---

![Linearize](/images/brink_flow_step4.svg)

In this step, we do a depth first walk of the AST to produce a linear sequence of operations.  The depth first walk ensures that dependencies are calculated before dependent operations.  For example, in the toy AST in the diagram, we obviously cannot perform the `==` comparison if we haven't yet calculated the result of the addition.

Flattening operations can get complex.  I recommend deferring as much user error checking as possible to the IR phase.  Some of the activities in linearizing include:
* Inlining chunks of the AST as required
* Expanding a single AST token into multiple primitive operations
* Creating parameter lists for operations
* Creating temporary variables to hold intermediate results

Let's expand on the last two bullets.  Each _edge_ between nodes in the AST means we'll do one of two things:
* Add a parameter to an operation, e.g. `123` is a parameter of `plus`.
* Create a temporary variable, e.g. `temp0` holds the result of the `plus` and is an input to the `double_eq`.

---

![Create IR](/images/brink_flow_step5.svg)

Now we have a nice ordered sequence of _things_ from step 4, but they're half-baked and possibly contain user bugs that have gone undetected so far.  In this step we create a proper independent representation (IR) of the program.  As we convert each linear operation into an IR operation, the compiler checks that each operation is well formed, e.g. has the expected number of operations, has a sane types, etc.

---

![Create IR](/images/brink_flow_step6.svg)

By this point, we're deep in DSL specific territory and your plan may need to deviate from how Brink works.  So that you understand Brink's reference point, here are a couple important points:
* Unlike a C compiler, Brink does not generate executable machine code from the IR.  Instead, Brink executes its IR immediately in its own internal VM.
* Brink's VM needs a wrapper loop that _repeatedly_ executes the IR until all the internal characteristics (sizes, offsets, etc) of Brink's binary output stabilize.  Depending on your DSL, you may not need to worry about such a thing.

---

Banner photo by [Dan Meyers](https://unsplash.com/@dmey503?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/)
  



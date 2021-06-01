---
layout: post
title:  "Writing a Domain Specific Language in Rust, Part 5 - Building a Graph with an Arena"
date:   2021-05-30 10:00:00 -0800
categories: Rust
---

tl;dr - Part 5 is about avoiding pointer and reference complications with an arena pattern. Back to [Part 4]({% link _posts/2021-04-26-writing_a_dsl_in_rust_part_4.markdown %})

![Cold Play at the Moda Center in 2017](/images/cold_play_moda_center.jpg)
# Rocking the Arena
For those of us coming from a C/C++ background, we know data structures like trees just *want* to use pointers.  Pointers provide a concise physical representation of a logical *edge* connecting one tree *node* to another.  As a tree grows, we change existing pointers and add new ones, making liberal use of null pointers as needed.

Efficient as they are, building trees using pointery C/C++ notions does not work in safe Rust.  [Nick Cameron provides](https://github.com/nrc/r4cppp/blob/master/graphs/README.md) a detailed discussion of the trouble as regards graphs.  The problem boils down to Rusts strict reference rules blocking construction of a tree piece by piece.  For example, suppose we traverse from parent node to child node by getting a reference to the child.  Now the parent node and some local variable hold references, so we can't also get a mutable reference to perform an update in the child.  We'd be fine if trees popped into existence fully formed.

What to do?
 
We could use raw pointers and *unsafe* Rust, but tree structures are central to many compiler operations.  If we can't bring the value of safe Rust to the main mechanics of a program, what's the point of using Rust?

## The `Rc<RefCell<Node>>` Approach
We could use the [reference counted reference cell pattern](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html#having-multiple-owners-of-mutable-data-by-combining-rct-and-refcellt) and manage borrowing at runtime.  This approach works, but is slightly more cumbersome due to the nested reference counters.

Here's a short teaser for this style.

```Rust
use std::rc::Rc;
use std::cell::RefCell;

type NodeRef = Rc<RefCell<Node>>;

struct Node {
    name: String,
    edges: Vec<NodeRef>,
}

impl Node {
    fn new(name: String) -> NodeRef {
        Rc::new(RefCell::new(Node {
            name: name.to_owned(),
            edges: Vec::new(),
        }))
    }

    fn add_child(&mut self, n: &NodeRef) {
        self.edges.push(n.clone());
    }
}
```

A tidy working example is again from [Nick Cameron on github](https://github.com/nrc/r4cppp/blob/master/graphs/src/rc_graph.rs).

For building an abstract syntax tree, the practical downside to the `Rc<RefCell<Node>>` are the memory overheads for the bookkeeping and heap allocations.  Each allocation is technically another lifetime to worry about as well.  Here's an informative [debate on reddit](https://www.reddit.com/r/rust/comments/jkh99u/how_heavy_is_rcrefcellt/) if you want to dig in.

## Enter the Arena

IMHO, arenas provide the cleanest way to build trees in Rust.  __An arena is a group of objects *stored* in a linearly indexed container, but which have a more complex *logical* arrangement.__  Why go with an arena instead of `Rc<RefCell<Node>>`?
* All the objects in the arena have the same lifetime from Rust's point of view.  That's a much better match for our compiler use case than separately allocated heap objects.
* We can iterate over nodes in the arena independently of the tree structure for debug or bookkeeping.
* High quality crates exist if you don't want to roll your own.  I used [indextree](https://docs.rs/crate/indextree/4.3.1).

The concept is simple.  For example, the following diagram shows how a small tree might be arranged in an arena.

![Arena Concept](/images/arena_1.svg)

## What to Store in a Node?

Besides parent/child arrangement, [Brink](https://github.com/steveking-gh/brink) tracks the following token information in its abstract syntax tree.
* The [logos](https://docs.rs/logos) enum that matched the token
* The original source location span of the token
* The trimmed string representation of the token

In important cases such as error reporting, __we need the information above, but don't care about the parent child/relationships of the tree__.  Code paths don't want to unnecessarily hold a reference to an element in the arena.  To create this isolation, Brink stores most token information in a dedicated token vector.  This vector is just a regular Rust vector not associated with the arena.  The data payload Brink stores in the arena based tree nodes is just an index into this separate token vector.

![Token information outside the arena](/images/token_vec_1.svg)

To summarize, we've reduced the complexity of representing an abstract syntax tree into nothing but vector indices!



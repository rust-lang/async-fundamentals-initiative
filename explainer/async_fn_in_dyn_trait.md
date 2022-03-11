# Async fn in dyn trait

![planning rfc][]

{{#include ../badges.md}}

Welcome! This document explores how to combine `dyn` and `impl Trait` in return position. This is crucial pre-requisite for async functions in traits. As a motivating example, consider the trait `AsyncIterator`:

```rust
trait AsyncIterator {
    type Item;
    
    async fn next(&mut self) -> Option<Self::Item>;
}
```

The `async fn` here is, of course, short for a function that returns `impl Future`:

```rust
trait AsyncIterator {
    type Item;
    
    fn next(&mut self) -> impl Future<Output = Option<Self::Item>>;
}
```

The focus of this document is on how we can support `dyn AsyncIterator`. For an examination of why this is difficult, see [this blog post](https://smallcultfollowing.com/babysteps//blog/2021/09/30/dyn-async-traits-part-1/).

## Key details

Here is a high-level summary of the key details of our approach:

* Natural usage:
    * To use dynamic dispatch, just write `&mut dyn AsyncIterator`, same as any other trait.
    * Similarly, on the impl side, just write `impl AsyncIterator for MyType`, same as any other trait.
* Allocation by default, but not required:
    * By default, trait functions that return `-> impl Trait` will allocate a `Box` to store the trait, but only when invoked through a `dyn Trait` (static dispatch is unchanged).
    * To support no-std or high performance scenarios, types can customize how an `-> impl Trait` function is dispatch through `dyn`. We show how to implement an `InlineAsyncIterator` type, for example, that wraps another `AsyncIterator` and stores the resulting futures on pre-allocated stack space.
        * rust-lang will publish a crate, `dyner`, that provides several common strategies.
* Separation of concerns:
    * Users of a `dyn AsyncIterator` do not need to know (or care) whether the impl allocates a box or uses some other allocation strategy.
    * Similarly, authors of a type that implements `AsyncIterator` can just write an `impl`. That code can be used with any number of allocation adapters.


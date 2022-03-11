# How it works

![planning rfc][]

{{#include ../../badges.md}}

This section is going to dive into the details of how this proposal works within the compiler. As a user, you should not generally have to know these details unless you intend to implement your own adapter shims.

## Return to our running example

Let's repeat our running example trait, `AsyncIterator`:

```rust
trait AsyncIterator {
    type Item;
    
    async fn next(&mut self) -> Option<Self::Item>;
}

async fn count(i: &mut dyn AsyncIterator) -> usize {
    let mut count = 0;
    while let Some(_) = i.next().await {
        count += 1;
    }
    count
}
```

Recall that `async fn next` desugars to a regular function that returns an `impl Trait`:

```rust
trait AsyncIterator {
    type Item;
    
    fn next(&mut self) -> impl Future<Output = Option<Self::Item>>;
}
```

Which in turn desugars to a trait with an associated type (note: precise details here are still being debated, but this desugaring is "close enough" for our purposes here):

```rust
trait AsyncIterator {
    type Item;
    
    type next<'me>: Future<Output = Option<Self::Item>>
    fn next(&mut self) -> Self::next<'_>;
}
```

## Challenges of invoking an async fn through `dyn`

When `count` is compiled, it needs to invoke `i.next()`. But the `AsyncIterator` trait simply declares that `next` returns some `impl Future` (i.e., "some kind of future"). Each impl will define its own return type, which will be some kind of special struct that is "sufficiently large" to store all the state that needed by that particular impl.

When an async function is invoked through static dispatch, the caller knows the exact type of iterator it has, and hence knows exactly how much stack space is needed to store the resulting future. When `next` is invoked through a `dyn AsyncIterator`, however, we can't know the specific impl and hence can't use that strategy. Instead, what happens is that invoking `next` on a `dyn AsyncIterator` *always* yields a *pointer* to a future. Pointers are always the same size no matter how much memory they refer to, so this strategy doesn't require knowing the "underlying type" beneath the `dyn AsyncIterator`.

Returning a pointer, of course, requires having some memory to point at, and that's where things get tricky. The easiest (and default) solution is to allocate and return a `Pin<Box<dyn Future>>`, but we wish to support other kinds of pointers as well (e.g., pointers into pre-allocated stack space). We'll explain how our system works in stages, first exploring a version that hardcodes the use of `Box`, and then showing how that can be generalized.

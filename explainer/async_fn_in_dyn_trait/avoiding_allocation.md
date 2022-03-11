# Using dyn without allocation

![planning rfc][]

{{#include ../../badges.md}}

In the previous chapter, we showed how you can invoke async methods from a `dyn Trait` value in a natural fashion. In those examples, though, we assume that it was ok to allocate a `Box` for every call to an async function. For most applications, this is true, but for some applications, it is not. This could be because they intend to run in a kernel or embedded context, where no allocator is available, or it could be because of a very tight loop in which allocation introduces too much overhead. The good news is that our design allows you to avoid using Box, though it does take a bit of work on your part. 

In general, functions that accept a `&dyn Trait` as argument don't control how memory is allocated. So the `count` function that we saw before can be used equally well on a no-std or kernel platform:

```rust
async fn count(iter: &mut dyn AsyncIterator) {
    // Whether or not `iter.next()` will allocate a box
    // depends on the underlying type; this `count` fn
    // doesn't have to know, and so it works equally
    // well in a no-std or std environment.
    ...
}
```

The decision about whether to use box or some other way of returning a future is made by the type implementing the async trait. In [the previous example](./how_it_feels.md), the type was `YieldingRangeIterator`, and its impl didn't make any kind of explicit choice, and thus the default is that it will allocate a `Box`:

```rust
impl AsyncIterator for YieldingRangeIterator {
    type Item = u32;

    async fn next(&mut self) {
        // The default behavior here is to allocate a `Box`
        // when `next` is called through a `dyn AsyncIterator`
        // (no `Box` is allocated when `next` is called through
        // static dispatch, in that case the future itself is
        // returned.)
        ...
    }
}
```

If you want to use `YieldingRangeIterator` in a context without `Box`, you can do that by wrapping it in an *adapter* type. This adapter type will implement an alternative memory allocation strategy, such as using pre-allocated stack storage.

For the most part, there is no need to implement your own adapter type, beacuse the `dyner` crate (to be published by rust-lang) includes a number of useful ones. For example, to pre-allocate the `next` future on the stack, which is useful both for performance or no-std scenarios, you could use an "inline" adapter type, like the `InlineAsyncIterator` type provided by the `dyner` crate:

```rust
use dyner::InlineAsyncIterator;
//         ^^^^^^^^^^^^^^^^^^^
//         Inline adapter type

async fn count_range(mut x: YieldingRangeIterator) -> usize {
    // allocates stack space for the `next` future:
    let inline_x = InlineAsyncIterator::new(x); 
    
    // invoke the `count` fn, which will no use that stack space
    // when it runs
    count(&mut inline_x).await
}
```

Dyner provides some other strategies, such as the `CachedAsyncIterator` (which caches the returned `Box` and re-uses the memory in between calls) and the `BoxInAllocatorAsyncIterator` (which uses a `Box`, but with a custom allocator).

## How you apply an existing "adapter" strategy to your own traits

The `InlineAsyncIterator` adapts an `AsyncIterator` to pre-allocate stack space for the returned futures, but what if you want to apply that inline stategy to one of your traits? You can do that by using the `#[inline_adapter]` attribute macro applied to your trait definition:

```rust
#[inline_adapter(InlineMyTrait)]
trait MyTrait {
    async fn some_function(&mut self);
}
```

This will create an adapter type called `InlineMyTrait` (the name is given as an argument to the attribute macro). You would then use it by invoking `new`:

```rust
fn foo(x: impl MyTrait) {
    let mut w = InlineMyTrait::new(x);
    bar(&mut w);
}

fn bar(x: &mut dyn MyTrait) {
    x.some_function();
}
```

If the trait is not defined in your crate, and hence you cannot use an attribute macro, you can use this alternate form, but it requires copying the trait definition:

```rust
dyner::inline::adapter_struct! {
    struct InlineAsyncIterator for trait MyTrait {
        async fn foo(&mut self);
    }
}
```

# How the running example could work with `Box`

![planning rfc][]

{{#include ../../badges.md}}

Before we get into the full system, we're going to start by *just* explaining how a system that hardcodes `Pin<Box<dyn Future>>` would work. In that case, if we had a `dyn AsyncIterator`, the vtable for that async-iterator would be a struct sort of like this:

```rust
struct AsyncIteratorVtable<I> {
    type_tags: usize,
    drop_in_place_fn: fn(*mut ()), // function that frees the memory for this trait
    next_fn: fn(&mut ()) -> Pin<Box<dyn Future<Output = Option<I> + '_>>
}
```

This struct has three fields:

* `type_tags`, which stores type information used for [`Any`](https://doc.rust-lang.org/std/any/trait.Any.html)
* `drop_in_place_fn`, a funcdtion that drops the memory of the underlying value. This is used when the a `dyn AsyncIterator` is dropped; e.g., when a `Box<dyn AsyncIterator>` is dropped, it calls [`drop_in_place`](https://doc.rust-lang.org/std/ptr/fn.drop_in_place.html) on its contents.
* `next_fn`, which stores the function to call when the user invokes `next`. You can see that this function is declared to return a `Pin<Box<dyn Future>>`.

(This struct is just for explanatory purposes; if you'd like to read more details about vtable layout, see [this description](https://rust-lang.github.io/dyn-upcasting-coercion-initiative/design-discussions/vtable-layout.html).)

Invoking `i.next()` (where `i: &mut dyn AsynIterator`) ultimately invokes the `next_fn` from the vtable and hence gets back a `Pin<Box<dyn Future>>`:

```rust
i.next().await

// becomes

let f: Pin<Box<dyn Future<Output = Option<I>>>> = i.next();
f.await
```

## How to build a vtable that returns a boxed future

We've seen how `count` calls a method on a `dyn AsyncIterator` by loading `next_fn` from the vtable, but how do we construct that vtable in the first place? Let's consider the struct `YieldingRangeIterator` and its `impl` of `AsyncIterator` that we saw before in an earlier section:


```rust
struct YieldingRangeIterator {
    start: u32,
    stop: u32,
}

impl AsyncIterator for YieldingRangeIterator {
    type Item = u32;

    async fn next(&mut self) {...}
}
```

There's a bit of a trick here. Normally, when we build the vtable for a trait, it points directly at the functions from the impl. But in this case, the function in the impl has a different return type: instead of returning a `Pin<Box<dyn Future>>`, it returns some `impl Future` type that could have any size. This is a problem. 

To solve it, the vtable doesn't directly reference the `next` fn from the impl, instead it references a "shim" function that allocates the box:

```rust
fn yielding_range_shim(
    this: &mut YieldingRangeIterator,
) -> Pin<Box<dyn Future<Output = Option<u32>>>> {
    Box::pin(<YieldingRangeIterator as AsyncIterator>::next(this))
}
```

This shim serves as an "adaptive layer" on the callee's side, converting from the `impl Future` type to the `Box`. More generally, we can consider the process of invoking a method through a dyn as having adaptation on both sides, like shown in this diagram:

![diagram](https://gist.githubusercontent.com/nikomatsakis/c0772e1827fd50e72c5052c8504b8a69/raw/4e943c7f762ec9ded11b5467ae952a70a5c4c24a/diagram.svg)

(This diagram shows adaptation happening to the arguments too; but for this part of the design, we only need the adaptation on the return value.)

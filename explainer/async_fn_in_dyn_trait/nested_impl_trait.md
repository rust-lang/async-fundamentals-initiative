# Nested `impl Trait` and dyn adaptation

We've seen that `async fn` desugars to a regular function returning `impl Future`. But what happens when we have *another* `impl Trait` inside the return value of the `async fn`?

```rust
trait DebugStream {
    async fn next(&mut self) -> Option<impl Debug + '_>;
}

impl DebugStream for Factory {
    async fn next(&mut self) -> Option<impl Debug + '_> {
        if self.done {
            None
        } else {
            Some(&self.debug_state)
        }
    }
}
```

## Trait desugaring

How does something like this desugar? Let's start with the basics...

The trait first desugars to:

```rust
trait DebugStream {
    fn next(&mut self) ->
        impl Future<Output = impl Debug + '_> + '_;
}
```

which further desugars to:

```rust
trait DebugStream {
    type next<'me>: Future<Output = impl Debug + 'me>
    //                                         ^^^^^
    // This lifetime wouldn't be here if not for
    // the `'_` in `impl Debug + '_`
    where
        Self: 'me;
    fn next(&mut self) -> Self::next<'_>;
}
```

which further desugars to:

```rust
trait DebugStream {
    type next<'me>: Future<Output = Self::next_0<'me>>
    where
        Self: 'me;
    type next_0<'a>: Debug
    //         ^^^^
    // This lifetime wouldn't be here if not for
    // the `'_` in `impl Debug + '_`
    where
        Self: 'a; // TODO is this correct?
    fn next(&mut self) -> Self::next<'_>;
}
```

As we can see, this problem is more general than `async fn`. We'd like a solution to work for any case of nested `impl Trait`, including on associated types.

## Impl desugaring

Pretty much the same as the trait.

The impl desugars to:

```rust
impl DebugStream for Factory {
    fn next(&mut self) ->
        impl Future<Output = Option<impl Debug + '_>> + '_
    {...}
}
```

which further desugars to:

```rust
impl DebugStream for Factory {
    type next<'me> =
        impl Future<Output = Option<impl Debug + 'me>> + 'me
    //                                         ^^^^^
    // This lifetime wouldn't be here if not for
    // the `'_` in `impl Debug + '_`
    where
        Self: 'me; // TODO is this correct?
    fn next(&mut self) -> Self::next<'me>
    {...}
}
```

which further desugars to:

```rust
impl DebugStream for Factory {
    type next<'me> =
        impl Future<Output = Option<Self::next_0<'me>>> + 'me
    where
        Self: 'me;
    type next_0<'a> = impl Debug + 'a
    //         ^^^^
    // This lifetime wouldn't be here if not for
    // the `'_` in `impl Debug + '_`
    where
        Self: 'a;
    fn next(&mut self) -> Self::next<'me>
    {...}
}
```

## Dyn adaptation

Let's start by revisiting out the "basic" case: returning regular old `impl Future`.

```rust
trait BasicStream {
    async fn next(&mut self) -> Option<i32>;
    // Desugars to:
    fn next(&mut self) -> impl Future<Output = Option<i32>>;
}

struct Counter(i32);
impl BasicStream for Counter {
    async fn next(&mut self) -> Option<i32> {...}
}
```

As we saw before, the compiler generates a shim for our type's `next` function:

```rust
// Pseudocode for the compiler-generated shim that goes in the vtable.
fn counter_next_shim(
    this: &mut Counter,
) -> dynx Future<Output = Option<i32>> {
    // We would skip boxing for #[dyn(identity)]
    let boxed = Box::pin(<Counter as BasicStream>::next(this));
    <dynx Future>::new(boxed)
}
```

Now let's attempt to do the same thing for our original example. Here it is from above:

```rust
trait DebugStream {
    async fn next(&mut self) -> Option<impl Debug + '_>;
}

impl DebugStream for Factory {
    fn next(&mut self) ->
        impl Future<Output = Option<impl Debug + '_>> + '_
    {...}
}
```

Generating the shim here is more complicated, because now it must do two layers of wrapping.

```rust
// Pseudocode for the compiler-generated shim that goes in the vtable.
fn factory_next_shim(
    this: &mut Counter,
) -> dynx Future<Output = Option<i32>> {
    let fut: impl Future<Output = Factory::next_0> =
        <Factory as DebugStream>::next(this);

    // We need to turn the above fut into:
    //     impl Future<Output = dynx Debug>
    // To do this, we need *another* shim...
    struct FactoryNextShim<'a>(Factory::next<'a>);
    impl<'a> Future for FactoryNextShim<'a> {
        type Output = Option<dynx Debug>;
        fn next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
            let ret = <Factory::next<'a> as Future>::poll(
                // This is always sound, probably
                pin_project!(self).0,
                cx,
            );
            match ret {
                Poll::Ready(Some(output)) => {
                    // We would skip boxing for #[dyn(identity)] on
                    // impl Future for Factory::next.. which means
                    // #[dyn(identity)] on the impl async fn?
                    // Or do we provide a way to annotate the
                    // future and `impl Debug` separately? TODO
                    //
                    // Why Box::new and not Box::pin like below?
                    // Because `Debug` has no `self: Pin` methods.
                    let boxed = Box::new(output);
                    Poll::Ready(<dynx Debug>::new(boxed))
                }
                Poll::Ready(None) | Poll::Pending => {
                    // No occurrences of `Output` in these variants.
                    Poll::Pending
                }
            }
        }
    }
    let wrapped = FactoryNextShim(fut);

    // We would skip boxing for #[dyn(identity)]
    // Why Box::pin? Because `Future` has a `self: Pin` method.
    let boxed = Box::pin(wrapped);
    <dynx Future>::new(boxed)
}
```

This looks to be a lot of code, but here's what it boils down to:

> For some `impl Foo<A = impl Bar>`,
>
> We generate a wrapper type of our outer `impl Foo` and implement the `Foo` trait on it. Our implementation forwards the methods to the actual type, takes the return value, and maps any occurrence of the associated type `A` in the return type to `dynx Bar`.

This mapping can be done structurally on the return type, and it benefits from all the flexibility of `dynx` that we saw before. That means it works for references to `A`, provided the lifetime bounds on the trait's `impl Bar` allow for this.

There are probably cases that can't work. We should think more about what those are.

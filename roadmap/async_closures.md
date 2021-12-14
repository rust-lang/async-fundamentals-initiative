# Async closures

## Impact

* Able to create async closures that work like ordinary closures but which can await values.
* Analogous traits to `Fn`, `FnMut`, `FnOnce`, etc
* Reconcile async blocks and async closures

## Design notes

Async functions need their own traits, analogous to `Fn` and friends:

```rust
#[repr(async_inline)]
trait AsyncFnOnce<A> {
    type Output;

    // Uh-oh! You can't encode these as `async fn` using inline async functions!
    async fn call(mut self, args: A) -> Self::Output;
}

#[repr(async_inline)]
trait AsyncFnMut: AsyncFnOnce {
    type Output;

    async fn call_mut(&mut self, args: A) -> Self::Output;
}

#[repr(async_inline)]
trait AsyncFn: AsyncFnMut {
    // Uh-oh! You can't encode these as `async fn` using inline async functions!
    async fn call(&self, args: A) -> Self::Output;
}
```

Some notes:

`AsyncFnOnce` is almost the same as `Future`/`Async` -- both represent, effectively, a future that can be driven exactly once. The difference is that your type can distinguish statically between the uncalled state and the persistent state after being called, if you wish, by using separate types for each. This can be useful for situations where an `async fn` is `Send` up until the point it is called, at which point it creates inner state that is not `Send`.

The concept of `AsyncFn` is more clear, but it requires storing the state externally to make sense: how else can there be multiple parallel executions.

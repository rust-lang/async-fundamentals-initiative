# Async closures

## Impact

* Able to create async closures that work like ordinary closures but which can await values.
* Analogous traits to `Fn`, `FnMut`, `FnOnce`, etc
* Reconcile async blocks and async closures

## Design notes

The fundamental problem async closures are meant to solve is that **normal closures can't return a value that borrows from the closure itself** (or, by extension, anything it captures). This is a big problem in async because the execution of all async code is encapsulated in a future returned by our function. Since that asynchronous code often needs to operate on captured values, it must in turn borrow from the closure.

This blog post describes the problem in more detail: https://smallcultfollowing.com/babysteps/blog/2023/03/29/thoughts-on-async-closures/.

One of the assertions made by this post is that async functions need their own traits, analogous to `Fn` and friends:

```rust
trait AsyncFnMut<A> {
    type Output;
    async fn call(&mut self, args: A) -> Self::Output;
}

// Similarly for AsyncFn, AsyncFnOnce.
```

A generalization of this would be to define "lending function" traits.

```rust
trait LendingFnMut<A> {
    type Output<'this>
    where
        Self: 'this;
    
    fn call(&mut self, args: A) -> Self::Output<'_>;
    //      ^                                  ^^^^
    // Lends data from `self` as part of return value.
}
```

This trait is similar to what `AsyncFnMut` above desugars to, except without saying anything about async or futures.

The `LendingFnMut` trait is from a follow up post that explains how we can actually extend the *existing* `Fn` traits to support "lending functions" that can do exactly what we want. It takes advantage of the fact that existing Fn trait bounds must use a special `Fn(A, B) -> C` syntax that always specifies their output type.

https://smallcultfollowing.com/babysteps/blog/2023/05/09/giving-lending-and-async-closures/

Other notes:

`AsyncFnOnce` is almost the same as `Future`/`Async` -- both represent, effectively, a future that can be driven exactly once. The difference is that your type can distinguish statically between the uncalled state and the persistent state after being called, if you wish, by using separate types for each. This can be useful for situations where an `async fn` is `Send` up until the point it is called, at which point it creates inner state that is not `Send`.

The concept of `AsyncFn` is more clear, but it requires storing the state externally to make sense: how else can there be multiple parallel executions.

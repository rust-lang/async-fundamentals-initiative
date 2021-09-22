# Bounding async drop

As a special case of the [bounding futures] problem, we must consider `AsyncDrop`.

[bounding futures]: ./bounding_futures.md

```rust
async fn foo<T>(t: T) {
    runtime::sleep(22).await;
}
```

The type of `foo(t)` is going to be a future type like `FooFuture<T>`. This type will also include the types of all futures that get awaited (e.g., the return value of `runtime::sleep(22)` in this case). But in the case of `T`, we don't yet know what `T` is, and if it should happen to implement `AsyncDrop`, then there is an ["implicit await"](./implicit_await_with_async_drop.md) of that future. We have to ensure that the contents of that future are taken into account when we determine if `FooFuture<T>: Send`.
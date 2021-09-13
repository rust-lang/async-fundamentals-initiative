# 🔬 Evaluation

> The *evaluation* surveys the various design approaches that are under consideration.
It is not required for all initiatives, only those that begin with a problem statement
but without a clear picture of the best solution. Often the evaluation will refer to topics
in the [design-discussions] for more detailed consideration.

[design-discussions]: ./design-discussions/README.md

## Goals

The goal of the impact is to enable users to write `async fn` in traits and impls in a natural way. As a simple example, we would like to support the ability to write an `async fn` in any trait:

```rust
trait Connection {
    async fn open(&mut self);
    async fn send(&mut self);
    async fn close(&mut self);
}
```

Along with the corresponding impl:

```rust
impl Connection for MyConnection {
    async fn open(&mut self) {
        ...
    }

    async fn send(&mut self) {
        ...
    }

    async fn close(&mut self) {
        ...
    }
}
```

The goal in general is that `async fn` can be used in traits as widely as possible:

* for foundational traits, like reading, writing, and iteration;
* for async closures;
* for async drop, which is built in to the language;
* in `dyn` values, which introduce some particular complications;
* in libraries, for all the usual reasons one uses traits;
* in ordinary programs, using all manner of executors.

## Key challenges

There are a few key challenges to overcome in the design:

* Bounding the futures that are returned by `Send`, `'static`, or other traits.
* Supporting `dyn Trait` when `Trait` has `async fn`, and giving the user control over some of the tradeoffs involved.


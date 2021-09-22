# Dyn traits

Supporting `dyn Trait` when `Trait` contains an `async fn` is challenging:

```rust
trait Trait {
    async fn foo(&self);
}

impl Trait for TypeA {
    async fn foo(&self);
}

impl Trait for TypeB { ... }
```

Consider the desugared form of this trait:

```rust
trait Trait {
    type Foo<'s>: Future<Output = ()> + 's;

    fn foo(&self) -> Self::Foo<'_>;
}

impl Trait for TypeA {
    type Foo<'s> = impl Future<Output = ()> + 's;

    fn foo(&self) -> Self::Foo<'_> {
        async move { ... } // has some unique future type F_A
    }
}

impl Trait for TypeB { ... }
```

The primary challenge to using `dyn Trait` in today's Rust is that **`dyn Trait` today must list the values of all associated types**. This means you would have to write `dyn for<'s> Trait<Foo<'s> = XXX>` where `XXX` is the future type defined by the impl, such as `F_A`. This is not only verbose (or impossible), it also uniquely ties the `dyn Trait` to a particular impl, defeating the whole point of `dyn Trait`.

For this reason, the `async_trait` crate models all futures as `Box<dyn Future<...>>`:

```rust
#[async_trait]
trait Trait {
    async fn foo(&self);
}

// desugars to

trait Trait {
    fn foo(&self) -> Box<dyn Future<Output = ()> + Send + '_>;
}
```

This compiles, but it has downsides:

* Allocation is required, *even when not using dyn Trait*.
* The user must state up front whether `Box<dyn Future...>` is `Send` or not.
    * In `async_trait`, this is declared by writing `#[async_future(?Send)]` if desired.

## Desiderata

Here are some of the general constraints:

* The ability to use `async fn` in a trait without allocation
* When using a `dyn Trait`, the type of the future must be the same for all impls
    * This implies a `Box` or other pointer indirection, or something like [inline async fn](../design/inline_async_fn.md).
* It would be nice if it were possible to use `dyn Trait` in an embedded context (without access to `Box`)
    * This will not be possible "in general", but it could be possible for particular traits, such as `AsyncIterator`
# Async fns in traits

*A user's guide from the future.*

Updated: Sep 12, 2022

<!-- Original: https://hackmd.io/9AH8Zr9ESMmur0n6Nix96w?view -->

This document explains the proposed design for async functions in traits, from the point of view of a "knowledgeable Rust user". It is meant to cover all aspects of the design we expect people to have to understand to leverage this feature. It doesn't go into every detail of how it will be implemented.

## Intro

This guide explains how traits work when they contain async functions. For the most part, a trait that contains an async functions works like any other trait -- but there are a few interesting questions and interactions that come up with async functions that don't arise with ordinary functions. We'll use a series of examples to explain how everything works.

## Traits and impls with async fn syntax

Defining a trait with an async function is straightforward ("just do it"):

```rust
use std::io::Result;

struct ResourceId { }
struct ResourceData { }

trait Fetch {
    async fn fetch(&mut self, request: ResourceId) -> Result<ResourceData>;
}
```

Similarly, implementing such a trait is straightforward ("just do it"):

```rust
struct HttpFetch { }

impl Fetch for HttpFetch {
    async fn fetch(&mut self, request: ResourceId) -> Result<ResourceData> {
        ... /* something that may await */ ...
    }
}
```

## Writing generic functions using async traits

Generic functions, types, etc that reference async traits work just like you would expect from other traits:

```rust
async fn fetch_and_process(f: impl Fetch, r: ResourceId) -> Result<()> {
    let data: ResourceData = f.fetch(r).await?;
    process(data)?;
    Ok(())
}
```

That example used [`impl Trait`](https://rust-lang.github.io/impl-trait-initiative/explainer/apit.html), but of course one could expand it to the (mostly equivalent) version with an explicit generic type:

```rust
async fn fetch_and_process<F>(f: F, r: ResourceId) -> Result<()>
where
    F: Fetch,
{
    let data: ResourceData = f.fetch(r).await?;
    process(data)?;
    Ok(())
}
```


## Bounding with Send, Sync, and other bounds

Suppose that we wanted to create a version of `fetch_and_process` in which the "fetching and processing" took place in another task? We can write a variant of `fetch_and_process`, `fetch_and_process_in_task`, that does this, but we have to add new where-clauses:

```rust
async fn fetch_and_process_in_task<F>(
    f: F,
    r: ResourceId,
)
where
    F: Fetch,
    F: Send + 'static, // 👈 Added so `F` can be sent to new task
    F::fetch(..): Send // 👈 Added, this syntax is new!
                       //    We will be explaining it shortly.
{
    tokio::spawn(async move {
        let data: ResourceData = f.fetch(r).await?;
        process(data)?;
        Ok(())
    });
}
```

If you prefer, you could write it with `impl Trait` syntax like so:


```rust
async fn fetch_and_process_in_task(
    f: impl Fetch<fetch(..): Send> + Send + 'static,
    //            ~~~~~~~~~~~~~~~    ~~~~~~~~~~~~~~
    //            Added! Leverages the new `fetch(..)`
    //            syntax we are about to explain, but
    //            also associated type bounds (RFC 2289).
    r: ResourceId,
) {
    tokio::spawn(async move {
        let data: ResourceData = f.fetch(r).await?;
        process(data)?;
        Ok(())
    });
}
```

Let's walk through those where-clauses in detail:

* `F: Send` and `F: 'static` -- together, these indicate that it is safe to move `F` over to another task. The `Send` bound means that `F` doesn't contain any thread-local types (e.g., `Rc`), and the `'static` means that `F` doesn't contain any borrowed data from the current stack.
* `F::fetch(..): Send` -- this is new syntax. It says that "the type returned by a call to `fetch()` is `Send`". In this case, the type being returned is a future, so this is saying "the future returned by `F::fetch()` will be `Send`".

## Function call bounds in more detail

The `F::fetch(..): Send` bound is actually an example of a much more general feature, return type notation. This notation lets you refer to the type that will be returned by a function call in all kinds of places.

As a simple example, you could now write a type like `identity_fn(u32)` to mean "the type returned by `identity_fn` when invoked with a `u32`":

```rust
let f: identity_fn(u32) = identity_fn(22_u32);
```

Knowing the types of arguments can be important for figuring out the return type! For example, if `identity_fn` is defined like so...

```rust
fn identity_fn<T>(t: T) -> T {
    t
}
```

...then `identity_fn(u32)` is equivalent to `u32`, `identity_fn(i32)` is equivalent to `i32`, and so on.


When you are using return type notation in a bound, you can use `..` to mean "for all the types accepted by the function". So when we write `F::fetch(..): Send`, that is in fact equivalent to this much more explicit bound:

```rust
where
    for<'a> F::fetch(&'a mut F, ResourceData): Send
    // --> equivalent to `F::fetch(..): Send`
```

The more explicit version spells out explicitly that we want "the type returned by `Fetch` when given some `&'a mut F` and a `ResourceData`".

The `..` notation can only be used in bounds  because it refers to a whole range of types. If you were to write `let f: F::fetch(..) = ..`, that would mean that `f` has multiple types, and that is not currently permitted.

### Turbofish and return type notation

In some cases, specifying the types of the arguments to the function are not enough to uniquely specify its behavior:

```rust
fn make_default<T: Default>() -> Option<T> {
    Some(T::default())
}
```

Just as in an expression, you can use turbofish to handle scenarios like this. The following function references `make_default::<T>()`, for example, as a (rather convoluted) synonym for `Option<T>`:

```rust
fn foo<T>(t: T)
where
    make_default::<T>(): Send,
```

## Dynamic dispatch and async functions

The other major way to use traits is with `dyn Trait` notation. This mostly works normally with async functions, but there are a few differences to be aware of.

Let's start with a variant of the `fetch_and_process` function that uses `dyn Trait` dispatch:

```rust
async fn fetch_and_process_dyn(f: &mut dyn Fetch, r: ResourceId) -> Result<()> {
    let data: ResourceData = f.fetch(r).await?;
    process(data)?;
    Ok(())
}
```

Looks simple! But what, what about the code that calls `fetch_and_process_dyn`, how does that look? This is where things with async functions get a bit more complicated. The obvious code to call `fetch_and_process_dyn`, for example, will not compile:

```rust
async fn caller() {
    let mut f = HttpFetch { .. };
    fetch_and_process_dyn(&mut f); // 👈 ERROR!
}
```

Compiling this we get the following message:

```
error[E0277]: the type `HttpFetch` cannot be converted to a
              `dyn Fetch` without an adapter
 --> src/lib.rs:3:23
  |
3 |     fetch_and_process_dyn(&mut f);
  |     --------------------- ^^^^^^ the trait `Foo` is not implemented for `HttpFetch`
  |     |
  |     required by a bound introduced by this call
  |
  = help: consider introducing the `Boxing` adapter,
    which will box the futures returned by each async fn
3 |     fetch_and_process_dyn(Boxing::new(&mut f));
  |                           ++++++++++++      +
```

### The `Boxing` adapter

What is going on here? The `Boxing` adapter indicates that, each time you invoke an `async fn` through the `dyn Fetch`, the future that is returned will be boxed.

The reasons that boxing are required here are rather subtle. The problem is that the amount of memory a future requires depends on the details of the function it results from (i.e., it is based on how much state is preserved across each `await` call). Normally, we store the data for futures on the stack, but this requires knowing precisely which function is called so that we can know exactly how much space to allocate. But when invoking an async function through a `dyn` value, we can't know what type is behind the `dyn`, so we can't know how much stack space to allocate.

To solve this, Rust only permits you to create a `dyn` value if the size of each future that is returned is the same size as a pointer. The `Boxing` adapter ensures this is the case by boxing each of the futures into the heap instead of storing their data on the stack.

:::info
**What if I don't want to box my futures?**

For the vast majority of applications, `Boxing` works great. In fact, the `#[async_trait]` procedural macro that most folks are using today boxes *every* future (the built-in design only boxes when using dynamic dispatch).

However, there are times when boxing isn't the right choice. Particularly tight loops, for example, or in `#[no_std]` scenarios like kernel modules or IoT devices that lack an operating system. For those cases, there are other adapters available that allocate futures on the stack or in other places. See [Appendix B](#Appendix-B-Other-adapters) for the details.
:::

### Specifying that methods return `Send`

Circling back to our `fetch_and_process` example, there is another potential problem. When we write this...

```rust
async fn fetch_and_process(f: &mut dyn Fetch, r: ResourceId) -> Result<()> {
    let data: ResourceData = f.fetch(r).await?;
    process(data)?;
    Ok(())
}
```

...this code will compile just fine, but if you are using a "work stealing runtime" (e.g., tokio by default), you may find that it doesn't work for you. The problem is that `f.fetch(r)` returns "some kind of future", but we don't know that the future is `Send`, and hence the `fetch_and_process()` future is itself not `Send`.

When using a `F: Fetch` argument, this worked out ok because we knew the type `F` exactly, and the future we returned would be `Send` as long as `F` was. With `dyn Fetch`, we don't know exactly what kind of type we will have, and we can only go based on what the type says -- and the type doesn't promise `Send` futures.

There are a few ways to write `fetch_and_process` such that the returned future is known to be `Send`. One of them is to make use of the return type bounds. Instead of writing `dyn Fetch`, we can write `dyn Fetch<fetch(..): Send>`, indicating that we need more than just "some `Fetch` instance" but "some `Fetch` instance that returns `Send` futures":

```rust
async fn fetch_and_process(
    f: &mut dyn Fetch<fetch(..): Send>, // 👈 New!
    r: ResourceId,
) -> Result<()> {
    let data: ResourceData = f.fetch(r).await?;
    process(data)?;
    Ok(())
}
```

Writing `dyn Fetch<fetch(..): Send>` is ok if you don't have to do it a lot, but if you do, you'll probably find yourself wishing for a shorter version. One option is to define a type alias:

```rust
type DynFetch<'bound> = dyn Fetch<fetch(..): Send> + 'bound;

async fn fetch_and_process(
    f: &mut DynFetch<'_>, // 👈 Shorter!
    r: ResourceId,
) -> Result<()> {
    let data: ResourceData = f.fetch(r).await?;
    process(data)?;
    Ok(())
}
```

## Appendix A: manual desugaring

In general, an async function in Rust can be desugared into a sync function that returns an `impl Future`. The same is true of traits. We could substitute the following definitions for the traits or impls and they would still compile:

```rust
trait Fetch {
    fn fetch(
        &mut self,
        request: ResourceId
    ) -> impl Future<Output = Result<ResourceData>> + '_;
}

impl Fetch for HttpFetch {
    fn fetch(
        &mut self,
        request: ResourceId
    ) -> impl Future<Output = Result<ResourceData>> + '_ {
        async move { ... /* something that may await */ ... }
    }
}
```

In fact, we can even "mix and match", using an async function in the trait and an `impl Trait` in the impl or vice versa.

It is also possible to write an impl with a specific future type:

```rust
impl Fetch for HttpFetch {
    #[refine]
    fn fetch(
        &mut self,
        request: ResourceId
    ) -> Box<dyn Future<Output = Result<ResourceData>> {
        Box::pin(async move {  /* something that may await */ })
    }
}
```

This last case represents a *refinement* on the trait -- the trait promises only that you will return "some future type". The impl is then promising to return a very specific future type, `Box<dyn Future>`. The `#[refine]` attribute indicates that detail is important, and that when people are referencing the return type of the `fetch()` method for the type `HttpFetch`, they can rely on the fact that it is `Box<dyn Future>` and not some other future type. (See [RFC 3245](https://github.com/rust-lang/rfcs/pull/3245) for more information.)

## Appendix B: Other adapters

### Inline adapter

The "inline" adapter allocates stack space in advance for each possible `async fn` in the trait. Unlike `Boxing`, it has a few sharp edges, and doesn't work with every trait. For this reason, it is not built-in to the compiler but rather shipped through a library.

To use the inline adapter, you first add the `dyner` crate to your package.

... TBW ...

## Appendix Z: Internal notes

*These are some notes and design questions that arose in reading and thinking through this document. They may be addressed in future revisions.*

Are there semver considerations in terms of when a `dyn` can be created? We should only permit creating a dyn when the impl has a `#[refine]` and the type implements `IntoDyn` or whatever trait we decide to use for the [`dyn*`](https://smallcultfollowing.com/babysteps/blog/2022/03/29/dyn-can-we-make-dyn-sized/) implementation.

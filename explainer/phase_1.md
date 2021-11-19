# Async Fundamentals Stage 1 Explainer

This document describes **Stage 1** of the "Async Fundamentals" plans. Our eventual goal is to make it so that, in short, wherever you write `fn`, you can also write `async fn` and have things work as transparently as possible. This includes in traits, even special traits like `Drop`.

Now that I've got you all excited, let me bring you back down to earth: That is our goal, but Stage 1 does not achieve it. However, we do believe that Stage 1 does enable async functions in traits in such a way that everything is possible, though it may not be easy.

The hope is that by having a stage 1 where stable Rust exposes the core functionality needed for async traits, we can get more data about how async traits will be used in practice. That can guide us as we try to resolve some of the (rather sticky) design questions that block making things more ergonomic. =)

## Review: how async fn works

When you write an async function in Rust:

```rust
async fn test(x: &u32) -> u32 {
    *x
}
```

This is actually shorthand for a function that returns an `impl Future` and which captures all of its arguments. The body of this function is an `async move` block, which simply creates and returns a future:

```rust
fn test<'x>(x: &'x u32) -> impl Future<Output = u32> + 'x {
    async move {
        *x
    }
}
```

The `await` operation can be performed on any value that implements `Future`. It causes the `async fn` to execute the future's "poll" function to see if it's value is ready. If not, then it suspends the current frame until it is re-invoked.

## Async fn in traits

Writing an async function in a trait desugars in just the same way as an async function elsewhere. Hence this trait:

```rust
trait HttpRequestProvider {
    async fn request(&mut self, request: Request) -> Response;
}
```

becomes:

```rust
trait HttpRequestProvider {
    fn request<'a>(&'a mut self, request: Request) -> impl Future<Output = Response> + 'a;
}
```

On stable Rust today, `impl Trait` is not permitted in "return position" within a trait, but we plan to allow it. It will desugar to a function that returns an associated type with the same name as the method itself:

```rust
trait HttpRequestProvider {
    type request<'a>: Future<Output = Response> + 'a;
    fn request<'a>(&'a mut self, request: Request) -> Self::request<'a>;
}
```

Calling `t.request(...)` thus returns a value of type `T::request<'a>`. We will reference this `request` variable later.

## Using async fn in traits

When you have `async fn` in a trait, you can use it with generic types in the usual way. For example, you could write a function that uses the above trait like so:

```rust
async fn process_request(
    mut provider: impl HttpRequestProvider,
    request: Request,
) {
    let response = provider.request(request).await;
    ...
}
```

Naturally you could also write the above example using explicit generics as well (just as with any impl trait):

```rust
async fn process_request<P>(
    mut provider: P,
    request: Request,
) where
    P: HttpRequestProvider,
{
    let response = provider.request(request).await;
    ...
}
```


## Naming the future that is returned

Like any function that returns an `impl Trait`, the return type from an `async fn` in a trait is anonymous. However, there are times when it can be very useful to be able to talk about it. One particular place where this comes up is when spawning tasks. Consider a function that invokes `process_request` (above) many times in parallel:

```rust
fn process_all_requests<P>(
    provider: P,
    requests: Vec<RequestData>,
) where
    P: HttpRequestProvider + Clone + Send,
{
    let handles =
        requests
            .into_iter()
            .map(|r| {
                let provider = provider.clone();
                tokio::spawn(async move {
                    process_request(provider, r).await;
                })
            })
            .collect();
    join_all(handles).await;
}
```

As is, compiling this function gives the following error, even though `P` is marked as `Send`:


```
error[E0277]: the future returned by `HttpRequestProvider::request` (invoked on `P`) cannot be sent between threads safely
   --> src/lib.rs:21:17
    |
21  |                 tokio::spawn(async move {
    |                 ^^^^^^^^^^^^ `<P as HttpRequestProvider>::request` cannot be sent between threads safely
    |
note: future is not `Send` as it awaits another future which is not `Send`
   --> src/lib.rs:35:5
    |
35  |     let response = provider.request(request).await?;
    |                    ^^^^^^^^^^^^^^^^^^^^^^^^^ this future is not `Send`
note: required by a bound in `tokio::spawn`
   --> /playground/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.13.0/src/task/spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ^^^^ required by this bound in `tokio::spawn`
```

The problem here is that, while `P: Send`, the *future* returned by `request` is not necessarily `Send` (it could, for example, create `Rc` state and store it in local variables). To fix this, one can add the bound on the `P::request` type:

```rust
fn process_all_requests<P>(
    provider: P,
    requests: Vec<RequestData>,
)
where
    P: HttpRequestProvider + Clone + Send,
    for<'a> P::request<'a>: Send,
{
    let handles =
        requests
            .into_iter()
            .map(|r| {
                let provider = provider.clone();
                your_runtime::spawn(async move {
                    process_request(provider, r).await;
                })
            })
            .collect();
    your_runtime::join_all(handles).await
}
```

The new bound is here:

```rust
    for<'a> P::request<'a>: Send,
```

The "higher-ranked" requirement says that "no matter what lifetime `provider` has, the result is `Send`". Specifying these higher-ranked lifetimes can be a bit cumbersome, and sometimes the GATs accumulate quite a large number of parameters. Therefore, the compiler also supports a shorthand form; if you leave off the GAT parameters, the `for<'a>` is assumed:

```rust
where
    P: HttpRequestProvider + Clone + Send,
    P::request: Send,
```

In fact, using a nightly feature, we can write this more compactly:

```rust
where
    P: HttpRequestProvider<request: Send> + Clone + Send,
```

### When would send bounds be required?

One thing we are not sure of is how often send bounds would be required in practice. The main point where such bounds are required are for generic async functions that will be "spawned". For example, the `process_request` function itself didn't require any kind of bounds on `request`:

```rust
async fn process_request(
    mut provider: impl HttpRequestProvider,
    request: Request,
) {
    let response = provider.request(request).await;
    ...
}
```

This is because nothing *in this function* was required to be `Send`. The problems only arise when you have a call to `spawn` or some other function that imposes a `Send` bound -- and even then, there may be no issue. For example, invoking `process_request` on a known type doesn't require any sort of where clauses:

```rust
your_runtime::spawn(async move {
    process_request(MyProvider::new(), some_request);
})
```

This works because the compiler is able to see that `process_request` is being called with a `MyProvider`, and hence it can determine exactly what future `process_request` will call when it invokes `provider.request`, and it can see that this future is `Send`.

The problems only arise when you invoke `spawn` on a value of a generic type, like `P` in our example above. In that case, the compiler doesn't know exactly what future will run, so it needs some bounds on `P::request`.

## Caveat: Async fn in trait are not dyn safe

There is one key limitation in stage 1: **traits that contain async fn are not dyn safe**. Instead, support for dynamic dispatch is provided via a procedural macro called `#[dyner]`. In future stages, we expect to support dynamic dispatch "natively", but we still need more experimentation and feedback on the requirements to find the best design.

The `#[dyner]` macro works as follows. You attach it to a trait:

```rust
#[dyner]
trait HttpRequestProvider {
    async fn request(&mut self, request: Request) -> Response;
}
```

It will generate, alongside the trait, a type `DynHttpRequestProvider` (`Dyn` + the name of the trait, in general):

```rust
// Your trait, unchanged:
trait HttpRequestProvider { .. }

// A struct to represent a trait object:
struct DynHttpRequestProvider<'me> { .. }

impl<'me> HttpRequestProvider for DynHttpRequestProvider<'me>
{ .. }
```

This type is a replacement for `Box<dyn HttpRequestProvider>`. You can create an instance of it by writing `DynHttpRequestProvider::from(x)`, where `x` is of some type that implements `HttpRequestProvider`. The `DynHttpRequestProvider` implements `HttpRequestProvider` but it forwards each method through using dynamic dispatch.

### Example usage

Suppose we had a `Context` type that was generic over an `HttpRequestProvider`:

```rust
struct Context<T: HttpRequestProvider> {
   provider: T,
}

async fn process_request(
    provider: impl HttpRequestProvider,
    request: Request,
) -> anyhow::Result<()> {
    let cx = Context {
        provider,
    };
    // ...
}
```

Now suppose that we wanted to remove the `T` type parameter, so that `Context` included a trait object. You could do this with `dyner` by altering the code as follows:

```rust
struct Context<'me> {
   provider: DynHttpRequestProvider<'me>,
}

async fn process_request(
    provider: impl HttpRequestProvider,
    request: Request,
) -> anyhow::Result<()> {
    let cx = Context {
        provider: DynHttpRequestProvider::new(provider),
    };
    // ...
}
```

You might be surprised to see the `'me` parameter -- this is needed because we don't whether the `provider` given to us includes borrowed data. If we knew that it had no references, we might also write:

```rust
struct Context {
   provider: DynHttpRequestProvider<'static>,
}

async fn process_request(
    provider: impl HttpRequestProvider + 'static,
    request: Request,
) -> anyhow::Result<()> {
    let cx = Context {
        provider: DynHttpRequestProvider::from(provider),
    };
    // ...
}
```

### Dyner with references

Dyner currently requires an allocator. The `DynHttpRequestProvider::new` method, for example, allocates a `Box` internally, and invoking an async function allocates a box to store the resulting future (the `#[async_trait]` crate does the same; the difference with this approach is that you only use the boxing when you are using dynamic dispatch).

Sometimes though we would like to construct our dynamic objects without using `Box`. This might be because we only have a `&T` access or simply because we don't need to allocate and would prefer to avoid the runtime overhead. In cases like that, you can use the `from_ref` and `from_mut` methods. `from_ref` takes a `&impl Trait` and gives you a `Ref<DynTrait>`; `Ref` is a special type that ensures you only use `&self` methods. `from_mut` works the same way but for `&mut Trait` types. Since the `provider` type never escapes `process_request`, we could rework our previous example to use mutable references instead of boxing:

```rust
struct Context<'me> {
   provider: &'me mut DynHttpRequestProvider<'me>,
}

async fn process_request(
    mut provider: impl HttpRequestProvider,
    request: Request,
) -> anyhow::Result<()> {
    let cx = Context {
        provider: DynHttpRequestProvider::from_mut(&mut provider),
    };
    // ...
}
```

The rest of the code would work just the same... you can still invoke methods in the usual way (e.g., `cx.provider.request(r).await`).

### Dyner and no-std

Because `dyner` requires an allocator, it is not currently suitable for no-std settings. We are exploring alternatives here: it's not clear if there is a suitable general purpose mechanism that could be used in settings like that. But part of the beauty of the dyner approach is that, in principle, there might be "dyner-like" crates that can be used in a no-std environment.

### Dyner all the things

Dyner is meant to be a "general purpose" replacement for `dyn Trait`. It expands the scope of what kinds of traits are dyn safe to include traits that use `impl Trait` in argument- and return-position. For example, the following trait is not dyn safe, but it is dyner safe:

```rust
#[dyner]
trait Print {
    fn print(&self, screen: &mut impl Screen);
    //                           ^^^^^^^^^^^
    //                    impl trait is not dyn safe;
    //                    it is dyner-safe if the trait
    //                    is also procssed with dyner
}

#[dyner]
trait Screen {
    fn output(&mut self, x: usize, y: usize, c: char);
}
```

### Dyner for external traits

For dyner to work well, all the traits that you reference from your dyner trait must be processed with dyner. But sometimes those traits may be out of your control! What can you do then? To support this, dyner permits you to apply dyner to an external trait definition. For example, support you had this trait, referencing the [`Clear` trait from `cc_traits`](https://docs.rs/cc-traits/0.7.1/cc_traits/trait.Clear.html):

```rust
#[dyner]
trait MyOp {
    fn apply_op(x: &mut impl cc_traits::Clear);
}
```

Applying `dyner` to `MyOp` will yield an error that "the type `cc_traits::DynClear` is not found". This is because the code that `dyner` generates expects that, for each `Foo` trait, there will be a `DynFoo` type available at the same location, and in this case there is not. You can fix this by using the `dyner::external_trait!` macro, but unfortunately doing so requires that you copy and paste the trait definition:

```rust
mod ext {
    dyner::external_trait! {
        pub trait cc_traits::Clear {
    	    fn clear(&mut self);
        }
    }
}
```

The `dyner::external_trait!` macro will generate two things:

* A re-export of `cc_traits::Clear`
* a type `DynClear`

Now we can rewrite our `MyOp` trait to reference `Clear` from this new location and everything works:

```rust
mod ext { ... }

#[dyner]
trait MyOp {
    fn apply_op(x: &mut impl ext::Clear);
    //                       ^^^ Changed this.
}
```

The `ext::Clear` path is just a re-exported version of `cc_traits::Clear`, so there is no change there, but the type `ext::DynClear` is well-defined.

### Dyner for things in the stdlib

The dyner crate already includes dyner-ified versions of things from the stdlib. However, to take advantage of them, you have to import the traits from `dyner::std`. For example, instead of referencing `std::iter::Iterator`, try `dyner::std::iter::Iterator`:

```rust
use dyner::std::iter::{Iterator, DynIterator};
//  ^^^^^^^                      ^^^^^^^^^^^

#[dyner]
trait WidgetStream {
    fn request(&mut self, r: impl Iterator<Item = String>);
    //                            ^^^^^^^^ the macro will look for DynIterator
}
```

You can use `use dyner::std::prelude::*` to get all the same traits as are present in the std prelude, along with their `Dyn` equivalents.

## Feedback requested

We'd love to hear what you think. Nothing here is set in stone, quite far from it!

Some questions we are specifically interested in getting answers to:

* What about this document was confusing to you? We want you to understand, of course, but we also want to improve the way we explain things.
* How often do you think you would use static vs dynamic dispatch in traits?
* How are you managing async fn in trait today? Do you think the `dyner` crate would work for you?
* How often do you think you use functions that require `Send`. Do you anticipate any problems from the bound scheme described above?

# Async Fundamentals Mini Vision Doc

> **WARNING:** This is an **archived** document, preserved for historical purposes. Go to the [explainer](../explainer.md) to see the most up-to-date material.

Grace and Alan are working at BoggleyCorp, developing a network service using async I/O in Rust. Internally, they use a trait to manage http requests, which allows them to easily change to different sorts of providers:

```rust
trait HttpRequestProvider {
    async fn request(&mut self, request: Request) -> anyhow::Result<Response>;
}
```

They start out using `impl HttpRequest` and things seem to be working very well:

```rust
async fn process_request(
    mut provider: impl HttpRequestProvider,
    request: Request,
) -> anyhow::Result<()> {
    let response = provider.request(request).await?;
    process_response(response).await;
}
```

As they refactor their system, though, they decide they would like to store the provider in a `Context` type. When they create the struct, though, they realize that `impl Trait` syntax doesn't work in that context:

```rust
struct Context {
   provider: impl HttpRequestProvider,
}

async fn process_request(
    mut provider: impl HttpRequestProvider,
    request: Request,
) -> anyhow::Result<()> {
    let cx = Context { 
        provider
    };
    // ...
}
```

Alan looks to Grace, "What do we do now?" Grace says, "Well, we could make an explicit type parameter, but I think a `dyn` might be easier for us here." Alan says, "Oh, ok, that makes sense." He alters the struct to use a `Box<dyn HttpRequestProvider>`:

```rust
struct Context {
   provider: Box<dyn HttpRequestProvider>,
}
```

However, this gets a compilation error: "traits that contain async fn are not dyn safe". The compiler does, however, suggest that they check out the experimental [`dyner` crate](https://github.com/nikomatsakis/dyner). The README for the crate advises them to decorate their trait with `dyner::make_dyn`, and to give a name to use for the "dynamic dispatch" type:

```rust
#[dyner::make_dyn(DynHttpRequestProvider)]
trait HttpRequestProvider {
    async fn request(&mut self, request: Request) -> anyhow::Result<Response>;
}
```

Following the readme, they also modify their `Context` struct like so:

```rust
struct Context {
   provider: DynHttpRequestProvider<'static>,
}

async fn process_request(
    provider: impl HttpRequestProvider,
    request: Request,
) -> anyhow::Result<()> {
    let cx = Context {
        provider: DynHttpRequestProvider::from(provider),
    };
    // ...
}
```

However, the code doesn't compile just as is. They realize that the `Context` is only being used inside `process_request` and so they follow the "time-limited" pattern of adding a lifetime parameter `'me` to the context. This represents the period of time in which the context is in use:

```rust
struct Context<'me> {
   provider: DynHttpRequestProvider<'me>,
}

async fn process_request(
    provider: impl HttpRequestProvider,
    request: Request,
) -> anyhow::Result<()> {
    let cx = Context {
        provider: DynHttpRequestProvider::from(provider),
    };
    // ...
}
```

At this point, they are able to invoke `provider.request()` as normal.

### Spawning tasks

As their project expands, Alan and Grace realize that they are going to have process a number of requests in parallel. They insert some code to spawn off tasks:

```rust
fn process_all_requests(
    provider: impl HttpRequestProvider + Clone,
    requests: Vec<RequestData>,
) -> anyhow::Result<()> {
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
    tokio::join_all(handles).await
}
```

However, when they write this, they get an error:

```
error: future cannot be sent between threads safely
   --> src/lib.rs:21:17
    |
21  |                 tokio::spawn(async move {
    |                 ^^^^^^^^^^^^ future created by async block is not `Send`
    |
note: captured value is not `Send`
   --> src/lib.rs:22:37
    |
22  |                     process_request(provider, r).await;
    |                                     ^^^^^^^^ has type `impl HttpRequestProvider + Clone` which is not `Send`
note: required by a bound in `tokio::spawn`
   --> /playground/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.13.0/src/task/spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ^^^^ required by this bound in `tokio::spawn`
help: consider further restricting this bound
    |
13  |     provider: impl HttpRequestProvider + Clone + std::marker::Send,
    |                                                +++++++++++++++++++
```

"Ah, right," thinks Alan. "I need to add `Send` to show that the provider is something we can send to another thread."

```rust
async fn process_all_requests(
    provider: impl HttpRequestProvider + Clone + Send,
    //                                           ^^^^ added this
    requests: Vec<Request>,
) {
    ...
}
```

But when he adds that, he gets *another* error. This one is a bit more complex:

```
error[E0277]: the future returned by `HttpRequestProvider::request` (invoked on `impl HttpRequestProvider + Clone + Send`) cannot be sent between threads safely
   --> src/lib.rs:21:17
    |
21  |                 tokio::spawn(async move {
    |                 ^^^^^^^^^^^^ `<impl HttpRequestProvider + Clone + Send as HttpRequestProvider>::Request` cannot be sent between threads safely
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
help: introduce a `provider: Send` bound to require this future to be send:
    |
13  ~     provider: impl HttpRequestProvider<request: Send> + Clone + Send,,
    |
```

Alan and Grace are a bit puzzled. They decide to to their friend Barbara, who knows Rust pretty well.

Barbara looks over the error. She explains to them that when they call an async function -- even in a trait -- that results in a future. "Yeah, yeah", they say, "we know that." Barbara explains that this means that saying that `provider` is `Send` does not necessarily imply that the future resulting from `provider.request()` is `Send`. "Ah, that makes sense," says Alan. "But what is this suggestion at the bottom?"

"Ah," says Barbara, "the notation `T: Foo<Bar: Send>` generally means the same as `T: Foo, T::Bar: Send`. In other words, it says that `T` implements `Foo` and that the associated type `Bar` is `Send`."

"Oh, I see," says Alan, "I remember reading now that when we use an `async fn`, the compiler introduces an associated type with the same name as the function that represents the future that gets returned."

"Right", answers Barbara. "So when you say `impl HttpRequestProvider<request: Send>` that means that the `request` method returns a `Send` value (a future, in this case)."

Alan and Grace change their project as the compiler suggested, and things start to work:

```rust
fn process_all_requests(
    provider: impl HttpRequestProvider<request: Send> + Clone,
    requests: Vec<RequestData>,
) -> anyhow::Result<()> {
    ...
}
```

### Dyner all the things (even non-async things)

As Alan and Grace get used to dyner, they start using it for all kinds of dynamic dispatch, including some code which is not async. It takes getting used to, but dyner has some definite advantages over the builtin Rust functionality, particularly if you use it everywhere. For example, you can use `dyner` with traits that have `self` methods and even methods which take and return `impl Trait` values (so long as those traits also use dyner, and they use the standard naming convention of prefixing the name of the trait with `Dyn`):

```rust
// The dyner crate re-exports standard library traits, along
// with Dyn versions of them (e.g., DynIterator). You do have
// to ensure that the `Iterator` and `DynIterator` are reachable
// from the same path though:
use dyner::iter::{Iterator, DynIterator};

#[dyner::make_dyn(DynWidgetStream)]
trait WidgetTransform {
    fn transform(
        mut self, 
        //  ^^^^ not otherwise dyn safe
        w: impl Iterator<Item = Widget>,
        // ^^^^^^^^^^^^^ this would not ordinarily be dyn-safe
    ) -> impl Iterator<Item = Widget>;
    //   ^^^^^^^^^^^^^ this would not ordinarily be dyn-safe
}
```

### Dyn trait objects for third-party traits

Using `dyner`, Alan and Barbara are basically unblocked. After a while, though, they hit a problem. One of their dependencies defines a trait that has no `dyn` equivalent (XXX realistic example?):

```rust
// In crate parser
trait Parser {
    fn parse(&mut self, input: &str);
}
```

They are able to manage this by declaring the "dyn" type themselves. To do so, however, they have to copy and paste the trait definition, which is kind of annoying:

```rust
mod parser {
    dyner::make_dyn_extern {
        trait parser::Parser {
            fn parse(&mut self, input: &str);
        }
    }
}
```

They can now use `crate::parser::{Parser, DynParser}` to get the trait and its Dyn equivalent; the `crate::parser::Parser` is a re-export. "Ah, so that's how dyner re-exports the traits from libstd", Grace realizes.

### What's missing?

* Spawning and needing `Send` bounds
* Appendix: how does dyner work?


### XXX

* replacement for `where Self: Sized`?
    * a lot of those problems go away *but*
    * we can also offer an explicit `#[optimization]` annotation:
        * causes the default version to be reproduced for dyn types and exempts the function from all limitations
* inherent async fn
* `impl A + B`
    * `DynerAB`-- we could even do that, right?
* 
* https://docs.rs/hyper/0.14.14/hyper/body/trait.HttpBody.html
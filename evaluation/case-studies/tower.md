# Tower Service Rework 

This case study presents how Tower would update it's core traits to improve 
ergonomics and useability with the aid of `async_fn_in_trait` (AFIT). 

## Background

[Tower] is a library of modular and reusable components for building robust 
networking clients and servers. The core use case of Tower is to enable users
to compose "stacks" of middleware in an easy and reusable way. To achieve this
currently Tower provide a [`Service`] and a [`Layer`] trait. The core
component is the `Service` trait as this is where all the async functionality
lives, as layers are not async.

Currently, the `Service` trait requires users to name their futures which does
not allow them to easily use `async fn` without boxing. In addition, borrowing
state within handlers is unergonomic due using associated types for the return
future type. 

The current `Service` trait looks something like this simplified version which
omits `poll_ready`, `Response/Error` handling:

```rust
trait Service<Request> {
    type Response;
    type Future: Future<Output = Self::Response>;

    fn call(&mut self, req: Request) -> Self::Future;
}
```

Middleware are `Service`'s that wrap another `S: Service` such that it will
internally call the inner service and wrap its behavior. This allows middleware
to modify the input type, wrap the output future and thus modify the output
type.

Due to `Service::call` requiring `&mut self` some middleware require their inner
`Service` to implement `Clone`. Unfortunetly, this bound has caused many issues
as it makes layering services quite complex due to the nested inference that
happens in [`ServiceBuilder`].

## Moving forward

One of the goals of moving to AFIT is to make using Tower easier and less error
prone. The current working idea that we have is to have a trait like:

```rust
trait Service<Request> {
    type Output;

    async fn call(&self, request: Request) -> Self::Response;
}
```

So far the experience has been pretty good working with AFIT with only a few
well known issues like dynamic dispatch and send bounds. In addition, we have
not looked at using `async-trait` because it forces `Send` bounds on our core 
trait which is an anti-goal for Tower as we would like to support both `Send`
and `!Send` services.

### Dynamic dispatch and Send 

Since, Tower stacks can become massive types and become very complicated for new
Rust users to work with we generally recommend boxing the Tower stack (a stack
is a group of Tower `Service`'s) after it has been constructed. This allows, for
example, clients to remove their need for generics and simplify their public api
while still leveraging Tower's composability. 

An example client:

```rust
struct Client {
    svc: BoxService<Request, Response, Error>,
}
```

Since, currently AFIT does not support dynamic traits we had to use a similar
work around to the Microsoft and Async Builder + Provider API case studies. This
workaround uses a second `DynService` trait that is object safe. Using this trait 
we can then define a `BoxService<'a, Req, Res, Err>` Tower service that completely 
erases the service type.

```rust
trait DynService<Req> {
    type Res;
    type Error;

    fn call<'a>(&'a self, req: Req) -> BoxFuture<'a, Result<Self::Res, Self::Error>>
    where
        Req: 'a;
}

pub struct BoxService<'a, Req, Res, Err> {
    b: Box<dyn 'a + DynService<Req, Res = Res, Error = Err>>,
}
```

This workaround works pretty well but falls short on one piece. The trait object
within `BoxService` is `!Send`. If we try to add `Send` bounds to the trait
object we eventually end up with a compiler error that says something like
`impl Future` doesn't implement `Send` and we end up at [rust-lang/rust#103854]

[rust-lang/rust#103854]: https://github.com/rust-lang/rust/issues/103854

#### RTN

Our issue above is that we have no way currently in Rust to express that some
async fn in a trait is `Send`. To achieve this, we tried out a [branch] from
@compiler-errors fork of rust. This branch implemented the return notation
syntax for expressing a bounds on impl trait based trait functions like async
fn. Using the synatx we were able to express this requirement to allow the trait
object to be sendable.

This blanket implementation allows us to have any `impl Service + Send`
automatically implement our dyn version of our `Service` trait.

```rust
impl<T, Req> DynService<Req> for T
where
    T: Service<Req, call(): Send> + Send,
    T::Res: Send,
    T::Error: Send,
    Req: Send,
{
    type Res = <T as Service<Req>>::Res;
    type Error = <T as Service<Req>>::Error;

    fn call<'a>(&'a self, req: Req) -> BoxFuture<'a, Result<Self::Res, Self::Error>>
    where
        Req: 'a,
    {
        Box::pin(self.call(req))
    }
}
```

In addition, one of the core requirements of tower is to be flexible and we
would like the core trait to be used in `Send` and `!Send` contexts. This means
we require that the way we express sendness is bounded at the users call site
rather than at the core trait level.

[branch]: https://github.com/compiler-errors/rust/tree/rtn

## Future areas of exploration

- We have not yet implemented the full stacking implementation that we currently
have with Tower. This means we have yet to potentailly run into any inference
issues that could happen when using the `Layer` trait from Tower.

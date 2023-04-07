# Tower AFIT Case Study 

This case study presents how Tower would update it's core traits to improve 
ergonomics and useability with the aid of `async_fn_in_trait` (AFIT). 

## Background

[Tower] is a library of modular and reusable components for building robust 
networking clients and servers. The core use case of Tower is to enable users
to compose "stacks" of middleware in an easy and reusable way. To achieve this
currently Tower provide a [`Service`] and a [`Layer`] trait. The core
component is the `Service` trait as this is where all the async functionality
lives. `Layer`s on the other hand are service constructors and allow users to
compose middleware stacks. Since, `Layer` does not do any async work it's trait
design is out of scope of this document for now.

Currently, the `Service` trait requires users to name their futures which does
not allow them to easily use `async fn` without boxing. In addition, borrowing
state within handlers is unergonomic due using associated types for the return
future type. This is because to allow the returned future to have a lifetime
it would require adding a lifetime generic associated type (GAT) which would force
an already verbose trait into an extremely verbose trait that is unergonomic for
users to implement. This thus requires all returned futures to be `'static`
which means that it must own all of its data. While this works right now its an
extra step that could be avoided by using `async fn`'s ability to produce
non-static futures in an ergonomic way.

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
to modify the input type, wrap the output future and modify the output type.

Due to `Service::call` requiring `&mut self` some middleware require their inner
`Service` to implement `Clone`. Unfortunetly, this bound has caused many issues
as it makes layering services quite complex due to the nested inference that
happens in [`ServiceBuilder`]. This has caused many massive and almost famous
error messages produced from Tower.

[Tower]: https://github.com/tower-rs/tower
[`Service`]: https://docs.rs/tower/latest/tower/trait.Service.html
[`ServiceBuilder`]: https://docs.rs/tower/latest/tower/struct.ServiceBuilder.html
[`Layer`]: https://docs.rs/tower/latest/tower/trait.Layer.html

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

One of the biggest wins we have experienced moving to AFIT has been how
ergonomic borrowing is within traits. For example, we have updated our retry
middleware to use AFIT. The implementation goes from two 100+ loc files to one
15 line implementation that is simpler to understand and verify. This large
change is due to not having to hand roll futures (which could be an artifact of
when tower was created) and the ability to immutably borrow the service via
`&self` rather than having to `Clone` the struct items. This is enabled by the
ergonomic borrows that AFIT provides.

### Dynamic dispatch and Send 

Since, Tower stacks can become massive types and become very complicated for new
Rust users to work with we generally recommend boxing the Tower stack (a stack
is a group of Tower `Service`'s) after it has been constructed. This allows, for
example, clients to remove their need for generics and simplify their public api
while still leveraging Tower's composability. 

An example client:

```rust
pub struct Client {
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

Another note, because `async fn` tend to eagerly capture items this new approach
requires that the `Requset` also be `Send`. This differs from how tower
currently works because to bound a returned future by `Send` does not require
that the input request type is `Send` as well since it does not lazily evaluate
it. That said, in practice, if you end up using something like [`tower::buffer`]
the `Request` type already needs to be `Send` to be able to send it over a
channel. This is likely a non-issue anyways since current consumer code tends to
use buffer to go from a `impl Service` to `impl Service + Clone`.

[branch]: https://github.com/compiler-errors/rust/tree/rtn
[`tower::buffer`]: https://docs.rs/tower/latest/tower/buffer/index.html

## Future areas of exploration

- We have not yet implemented the full stacking implementation that we currently
have with Tower. This means we have yet to potentailly run into any inference
issues that could happen when using the `Layer` trait from Tower.

## References

- https://github.com/LucioFranco/tower-playground/tree/lucio/sendable-box
- https://github.com/tower-rs/tower

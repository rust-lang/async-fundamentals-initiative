# Unresolved questions

![planning rfc][]

{{#include ../../badges.md}}

There are some interesting details that are yet to be resolved, and they become important indeed in the "feel" of the overall design. Covering those details however would make this document too long, so we're going to split it into parts. Nonetheless, for completeness, this section lists out some of those "questions yet to come".

## Which values that can be converted into a dynx struct?

In the previous section, we showed how a `#[dyn(identity)]` function must return "something that can be converted into a dynx struct", and we showed that a case of returning a `Pin<Box<impl Future, A>>` type. But what are the general rules for constructing a `dynx` struct? You're asking the right question, but that's a part of the design we haven't bottomed out yet. See [this design document](../../evaluation/design/dynx/creation.md) for more details.

## What about dyn with sendable future, how does that work?

We plan to address this in a follow-up RFC. The core idea is to build on the notation that one would use to express that you wish to have an async fn return a `Send`. As an example, one might write `AsyncIterator<next: Send>` to indicate that `next()` returns a `Send` future; when we generate the vtable for a `dyn AsyncIterator<next: Send>`, we can ensure that the bounds for `next` are applied to its return type, so that it would return a `dynx Future + Send` (and not just a `dynx Future`). We have also been exploring a more convenient shorthand for declaring "all the futures returned by methods in trait should be `Send`", but to avoid bikeshedding we'll avoid talking more about that in this document! See [this design document](../../evaluation/design/dynx/auto_trait.md) for more details.

## How do `dynx Trait` structs and "sealed traits" interact?

As described here, every dyn-safe trait `Trait` gets an "accompanying" `dynx Trait` struct and an `impl Trait for dynx Trait` impl for that struct. This can have some surprising interactions with unsafe code -- if you have a trait that can only be safely implemented by types that meet certain criteria, the impl for a `dynx` type may not meet those criteria. This can lead to undefined behavior. The question then is: *whose fault is that?* In other words, is it the language's fault, for adding impls you didn't expect, or the code author's fault, for not realizing those impls would be there (or perhaps for not declaring that their trait had additional safety requirements, e.g. by making the trait unsafe). This question turns out to be fairly complex, so we'll defer a detailed discussion beyond this summary. See [this design document](../../evaluation/design/dynx/sealed_traits.md) for more details.

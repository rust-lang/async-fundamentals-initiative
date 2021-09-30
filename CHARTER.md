# 📜 async fn fundamentals Charter
<!--
 Provide an introduction summarising the goals and motivation behind your
 initiative.
-->

This initiative is part of the wg-async-foundations [vision process].

## Proposal

`async fn` exists today, but does not integrate well with many core language features like traits, closures, and destructors. We would like to make it so that you can write async code just like any other Rust code.

## Goals

### Able to write `async fn` in traits and trait impls

The goal in general is that `async fn` can be used in traits as widely as possible:

* for foundational traits, like reading, writing, and iteration;
* for async closures;
* for async drop, which is built in to the language;
* in `dyn` values, which introduce some particular complications;
* in libraries, for all the usual reasons one uses traits;
* in ordinary programs, using all manner of executors.

#### Key outcomes

* [Async functions in traits][async trait] desugar to [impl Trait in traits]
* Traits that use `async fn` must still be [dyn safe][dyn async trait] though some tuning may be required
* Return futures must easily be [bound by `Send`][bounding]

### Support async drop

Users should be able to write ["async fn drop"][async drop] to declare that the destructor may await.

#### Key outcomes

* Types can perform async operations on cleanup, like closing database connections
* There's a way to detect and handle async drop types that are dropped synchronously
* Await points that result from async cleanup can be identified, if needed

### Support async closures

Support [async closures] and `AsyncFn`, `AsyncFnMut`, `AsyncFnOnce` traits.

#### Key outcomes

* Async closures work like ordinary closures but can await values
* Traits analogous to `Fn`, `FnMut`, `FnOnce` exist for async
* Reconcile async blocks and async closures

## Membership

| Role | Github |
| ---  | --- |
| [Owner] | [tmandry](https://github.com/tmandry) |
| [Liaison] | [nikomatsakis](https://github.com/nikomatsakis) |

[Owner]: https://lang-team.rust-lang.org/initiatives/process/roles/owner.html
[Liaison]: https://lang-team.rust-lang.org/initiatives/process/roles/liaison.html

[async drop]: ./design-discussions/async_drop.md
[async closures]: ./design-discussions/async_closures.md
[async trait]: ./design-discussions/static_async_trait.md
[bounding]: ./evaluation/challenges/bounding_futures.html
[dyn async trait]: ./design-discussions/dyn_async_trait.md
[impl Trait in traits]: ./design-discussions/impl_trait_in_traits.md
[vision process]: https://rust-lang.github.io/wg-async-foundations/vision.html

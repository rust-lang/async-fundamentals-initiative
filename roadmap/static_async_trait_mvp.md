# MVP: Static async fn in traits

This section defines an initial **minimum viable product** (MVP). This MVP is meant to be a subset of async fns in traits that can be implemented and stabilized quickly.

## In a nutshell

* In traits, `async fn foo(&self)` desugars to
    * an anonymous associated type `type Foo<'me>: Future<Output = ()>` (as this type is anonymous, users cannot actually name it; the name `Foo` here is for demonstrative purposes only)
    * a function `fn foo(&self) -> Self::Foo<'_>` that returns this future
* In impls, `async fn foo(&self)` desugars to
    * a value for the anonymous associated type `type Foo<'me> = impl Future<Output = ()>`
    * a function `fn foo(&self) -> Self::Foo<'_> { async move { ... } }`
* If the trait used `async fn`, then the impl must use `async fn` (and vice versa)
* Traits that use `async fn` are *not* dyn safe
    * In the MVP, traits using async fn can only be used with `impl Trait` or generics

## What this enables

* The MVP is sufficient for projects like [embassy], which already model async fns in traits in this way.
* TODO: Once we have a list of stakeholders, try to get a sense for how many uses of [`async-trait`] could be replaced

[embassy]: https://github.com/embassy-rs/embassy
[`async-trait`]: https://crates.io/crates/async-trait

## Notable limitations and workarounds

* No support for [`dyn`]
    * This is a fundamental limitation; the only workaround is to use [`async-trait`]
* No ability to name the resulting futures:
    * This means that one cannot build non-generic adapters that reference those futures.
    * Workaround: define a function alongside the impl and use a TAIT for its return type
* No ability to bound the resulting futures (e.g., to require that they are `Send`)
    * This rules out certain use cases when using work-stealing [executor styles], such as the [background logging] scenario. Note that many other uses of async fn in traits will likely work fine even with a work-stealing executor: the only limitation is that one cannot write generic code that invokes `spawn`.
    * Workaround: do the desugaring manually when required, which would give a name for the relevant future.

[background logging]: ../evaluation/scenarios/background-logging.md
[executor styles]: ../evaluation/executor-styles.md
[`dyn`]: ../evaluation/challenges/dyn_traits.md

## Implementation plan

* This MVP relies on having [generic associated types][gats] and [type alias impl trait][tait], but they are making good progress.
* Otherwise, the implementation is a straightforward desugaring, similar to how inherent async fns are implemented
* We may wish to also ship a variant of the [`async-trait`] macro that lets people easily experiment with this feature

[gats]: https://rust-lang.github.io/generic-associated-types-initiative/
[tait]: https://rust-lang.github.io/impl-trait-initiative/

## Forward compatibility

The MVP sidesteps a number of the [more challenging design problems][challenges]. It should be forwards compatible with:

* Adding support for [`dyn`] traits later
* Adding a mechanism to [bound the resulting futures][bound]
    * **WARNING:** It is NOT compatible with [Implied Send], however!
* Adding a mechanism to [name the resulting futures][name]
    * The futures added here are anonymous, and we can always add explicit names later.
    * *If* we were to [name the resulting futures after the methods][simple_names], and users had existing traits that used those same names already, this could present a conflict, but one that could be resolved.
* Supporting and bounding [async drop]
    * This trait will not exist yet with the MVP, and supporting `async fn` doesn't enable anything fundamental that we don't have to solve anyway.

[challenges]: ../evaluation/challenges.md
[bound]: ../evaluation/challenges/bounding_futures.md
[Implied Send]: ../evaluation/design/implied_send.md
[async drop]:  ../evaluation/challenges/bounding_async_drop.md
[name]: ../evaluation/challenges/naming_futures.md
[simple_names]: ../evaluation/design/simple_names.md

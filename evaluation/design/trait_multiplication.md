# Trait multiplication

## Status

Seems unlikely to be adopted, but may be the seed of a better idea

## Summary

Introduce a new form of bound, trait *multiplication*. One can write `T: Iterator * Send` and it means `T: Iterator<Item: Send> + Send` (using the notation from [RFC 2289]). More generally, `Foo * Bar` means `Foo + Bar` but also that `Foo::Assoc: Bar` for each associated type `Assoc` defined in `Foo` (including anonymous ones defined by async functions).

[RFC 2289]: https://rust-lang.github.io/rfcs/2289-associated-type-bounds.html

## What's great about this

The cool thing about this is that it means that **the check whether things are Send occurs exactly when it is needed, and not at the definition site**. This makes async functions in trait behave more analogously with ordinary impls. ([In contrast to implied bounds](./implied_send.html#not-analogous-to-async-fn-outside-of-traits).)

With this proposal, the [background logging] scenario would play out differently depending on the [executor style] being used:

[background logging]: ../scenarios/background-logging.md
[executor style]: ../executor-styles.md

* Thread-local: `logs: impl AsyncIterator<Item = String> + 'static`
* Thread per core: `logs: impl AsyncIterator<Item = String> + Send + 'static`
* Work stealing: `logs: impl AsyncIterator<Item = String> * Send + 'static`

The key observation here is that `+ Send` only tells you whether the *initial value* (here, `logs`) is `Send`. The `* Send` is needed to say "and the futures resulting from this trait are Send", which is needed in work-stealing sceanrios.

## What's not so great about this

### Complex

The distinction between `+` and `*` is subtle but crucial. It's going to be a new thing to learn and it just makes the trait system feel that much more complex overall.

### Reptitive for multiple traits

If you had a number of async traits, you would need `* Send` for each one:

```rust
trait Trait1 {
    async fn method1(&self);
}

trait Trait2 {
    async fn method2(&self);
}

async fn foo<T>()
where
    T: Send * (Trait1 + Trait2)
{

}
```

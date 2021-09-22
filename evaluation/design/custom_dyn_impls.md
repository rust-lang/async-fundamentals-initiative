# Custom dyn impls

As described in [dyn traits], `dyn Trait` types cannot include the types of each future without defeating their purpose; but outside of a `dyn` context, we *want* those associated types to have unique values for each impl. Threading this needle requires extending Rust so that the value of an associated type can be different for a `dyn Trait` and for the underlying impl.

[dyn traits]: ../challenges/dyn_traits.md

## How it works today

Conceptually, today, there is a kind of "generated impl" for each trait. This impl implements each method by indirecting through the vtable, and it takes the value of associated types from the dyn type:

```rust
trait Foo {
    type Bar;

    fn method(&self);
}

impl<B> Foo for dyn Foo<Bar = B> {
    type Bar = B;

    fn method(&self) {
        let f: fn(&Self) = get_method_from_vtable(self)
        f(self)
    }
}
```

Note that there are some known problems here, such as [soundness holes in the coherence check](https://github.com/rust-lang/lang-team/blob/master/design-meeting-minutes/2020-01-13-dyn-trait-and-coherence.md).

## Rough proposal

What we would like is the ability for this "dyn" impl to diverge more from the underlying impl. For example:

XXX to be continued

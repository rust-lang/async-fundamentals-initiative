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

Meanwhile, at the point where a type (say `u32`) is coerced to a `dyn Foo`, we generate a vtable based on the impl:

```rust
// Given
impl Foo for u32 {
    fn method(self: &u32) { XXX }
}

// we could a compile for `method`:
// fn `<u32 as Foo>::method`(self: &u32) { XXX }

fn eg() {
    let x: u32 = 22;
    &x as &dyn Foo // <-- this case
}

// generates a vtable with a pointer to that method:
//
// Vtable_Foo = [ ..., `<u32 as Foo>::method` ]
```

Note that there are some known problems here, such as [soundness holes in the coherence check](https://github.com/rust-lang/lang-team/blob/master/design-meeting-minutes/2020-01-13-dyn-trait-and-coherence.md).

## Rough proposal

What we would like is the ability for this "dyn" impl to diverge more from the underlying impl. For example, given a trait `Foo` with an `async` fn method:

```rust
trait Foo {
    async fn method(&self);
}
```

The compiler might generate an impl like the following:

```rust
impl<B> Foo for dyn Foo {
    //          ^^^^^^^ note that this type doesn't include Bar = ...

    type Bar = Box<dyn Future<Output = ()>>;
    //         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ because the result is hardcoded

    fn method(&self) -> Box<dyn Future<Output = ()>> {
        let f: fn(&Self) = get_method_from_vtable(self)
        f(self) 
    }
}
```

The vtable, meanwhile, resembles what we had before, except that it doesn't point directly to `<u32 as Foo>::method`, but rather to a wrapper function (let's call it `methodX`) that has the job of coercing from the concrete type into a `Box<dyn Future>`:

```
// Vtable_Foo = [ ..., `<u32 as Foo>::methodX`]
// fn `<u32 as Foo>::method`(self: &u32) { XXX  }
// fn `<u32 as Foo>::methodX`(self: &u32) -> Box<dyn> { Box::new(TheFuture)  }
```

## Auto traits

To handle "auto traits", we need multiple impls. For example, assuming we adopted [trait multiplication](./trait_multiplication.md), we would have multiple impls, one for `dyn Foo` and one for `dyn Foo * Send`:


```rust
trait Foo {
    async fn method(&self);
}

impl<B> Foo for dyn Foo {
    type Bar = Box<dyn Future<Output = ()>>;

    fn method(&self) -> Box<dyn Future<Output = ()>> {
            
    }
}

impl<B> Foo for dyn Foo * Send {
    //          ^^^^^^^^^^^^^^

    type Bar = Box<dyn Future<Output = ()> + Send>;
    //         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

    fn method(&self) -> Box<dyn Future<Output = ()>> {
            ....
    }
}

// compiles to:
//
// Vtable_Foo = [ ..., `<u32 as Foo>::methodX`]
// fn `<u32 as Foo>::method`(self: &u32) { XXX  }
// fn `<u32 as Foo>::methodX`(self: &u32) -> Box<dyn> { Box::new(TheFuture)  }
```

### Hard-coding box

One challenge is that we are hard-coding `Box` in the above impls. We could control this in a number of ways:

* Annotate the trait with an alternate wrapper type
* Extend `dyn` types with some kind of indicator of the wrapper (`dyn(Box)`) that they use for this case
* Generate impls for `Box<dyn>` -- has several shortcomings

### Applicable

Everything here is applicable more broadly, for example to types that return `Iterator`.

It'd be nice if we extended this capability of "writing your own dyn impls" to end-users.
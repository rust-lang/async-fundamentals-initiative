---
tags: draft-rfc
title: Return position `impl Trait` in traits
---

[![hackmd-github-sync-badge](https://hackmd.io/h9Cr4dfKR1KXC6P1lgM2cQ/badge)](https://hackmd.io/h9Cr4dfKR1KXC6P1lgM2cQ)


- Feature Name: return_position_impl_trait_in_traits
- Start Date: 2021-11-16
- RFC PR: [rust-lang/rfcs#3193](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)
- Initiative: [impl trait initiative](https://github.com/rust-lang/impl-trait-initiative)

# TODO

* [x] Relax dyn restriction for `where Self: Sized` methods
* [x] Describe interaction with `#[refine]`
* [x] Modify `async fn` in traits to allow `-> impl Future` in impls (must match exact desugaring without `#[refine]`)
* [ ] Flesh out motivation in contrast to associated types and ITIAT
* [ ] Mention why you'd use RPITIT so you don't have to name associated types with `dyn` (in the "why not named associated type" section / future possibilities section)

# Summary
[summary]: #summary

* Permit `impl Trait` in fn return position within traits and trait impls.
* This desugars to an anonymous associated type.

# Motivation
[motivation]: #motivation

The `impl Trait` syntax is currently accepted in a variety of places within the Rust language to mean "some type that implements `Trait`" (for an overview, see the [explainer] from the impl trait initiative). For function arguments, `impl Trait` is [equivalent to a generic parameter][apit] and it is accepted in all kinds of functions (free functions, inherent impls, traits, and trait impls). In return position, `impl Trait` [corresponds to an opaque type whose value is inferred][rpit]. In that role, it is currently accepted only in free functions and inherent impls. This RFC extends the support to cover traits and trait impls, just like argument position.

[explainer]: https://rust-lang.github.io/impl-trait-initiative/explainer.html
[apit]: https://rust-lang.github.io/impl-trait-initiative/explainer/apit.html
[rpit]: https://rust-lang.github.io/impl-trait-initiative/explainer/rpit.html

## Example use case

The use case for `-> impl Trait` in trait fns is similar to its use in other contexts: traits often wish to return "some type" without specifying the exact type. As a simple example that we will use through the RFC, consider the `NewIntoIterator` trait, which is a variant of the existing `IntoIterator` that uses `impl Iterator` as the return type:

```rust
trait NewIntoIterator {
    type Item;
    fn into_iter(self) -> impl Iterator<Item = Self::Item>;
}
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

*This section assumes familiarity with the [basic semantics of impl trait in return position][rpit].*

When you use `impl Trait` as the return type for a function within a trait definition or trait impl, the intent is the same: impls that implement this trait return "some type that implements `Trait`", and users of the trait can only rely on that. However, the desugaring to achieve that effect looks somewhat different than other cases of impl trait in return position. This is because we cannot desugar to a type alias in the surrounding module; we need to desugar to an associated type (effectively, a type alias in the trait).

Consider the following trait:

```rust
trait IntoIntIterator {
    fn into_int_iter(self) -> impl Iterator<Item = u32>;
}
```

The semantics of this are analogous to introducing a new associated type within the surrounding trait;

```rust
trait IntoIntIterator { // desugared
    type IntoIntIter: Iterator<Item = u32>;
    fn into_int_iter(self) -> Self::IntoIntIter;
}
```

(In general, this associated type may be generic; it would contain whatever generic parameters are captured per the generic capture rules given previously.)

This associated type is introduced by the compiler and cannot be named by users.

The impl for a trait like `IntoIntIterator` must also use `impl Trait` in return position:

```rust
impl IntoIntIterator for Vec<u32> {
    fn into_int_iter(self) -> impl Iterator<Item = u32> {
        self.into_iter()
    }
}
```

This is equivalent to the following "desugared" impl (except that the associated type is anonymous):

```rust
impl IntoIntIterator for Vec<u32> {
    type IntoIntIter = impl Iterator<Item = u32>;
    fn into_int_iter(self) -> Self::IntoIntIter {
        self.into_iter()
    }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Equivalent desugaring for traits

Each `-> impl Trait` notation appearing in a trait fn return type is desugared to an anonymous associated type; the name of this type is a fresh name that cannot be typed by Rust programmers. In this RFC, we will use the name `$` when illustrating desugarings and the like.

As a simple example, consider the following (more complex examples follow):

```rust
trait NewIntoIterator {
    type Item;
    fn into_iter(self) -> impl Iterator<Item = Self::Item>;
}

// becomes

trait NewIntoIterator {
    type Item;

    type $: Iterator<Item = Self::Item>;

    fn into_iter(self) -> <Self as NewIntoIterator>::$;
}
```

```rust
trait SomeTrait {
    fn method<P0, ..., Pm>()
}
```

## Equivalent desugaring for trait impls

Each `impl Trait` notation appearing in a trait impl fn return type is desugared to the same anonymous associated type `$` defined in the trait along with a function that returns it. The value of this associated type `$` is an `impl Trait`.

```rust
impl NewIntoIterator for Vec<u32> {
    type Item = u32;

    fn into_iter(self) -> impl Iterator<Item = Self::Item> {
        self.into_iter()
    }
}

// becomes

impl NewIntoIterator for Vec<u32> {
    type Item = u32;
    
    type $ = impl Iterator<Item = Self::Item>;

    fn into_iter(self) -> <Self as NewIntoIterator>::$ {
        self.into_iter()
    }
}
```


## Generic parameter capture and GATs

Given a trait method with a return type like `-> impl A + ... + Z` and an implementation of that trait, the hidden type for that implementation is allowed to reference:

* Concrete types, constant expressions, and `'static`
* `Self`
* Generics on the impl
* Certain generics on the method
    * Explicit type parameters
    * Argument-position `impl Trait` types
    * Explicit const parameters
    * Lifetime parameters that appear anywhere in `A + ... + Z`, including elided lifetimes

We say that a generic parameter is *captured* if it may appear in the hidden type. These rules are the same as those for `-> impl Trait` in inherent impls.

When desugaring, captured parameters from the method are reflected as generic parameters on the `$` associated type. Furthermore, the `$` associated type brings whatever where clauses are declared on the method into scope, excepting those which reference parameters that are not captured.

This transformation is precisely the same as the one which is applied to other forms of `-> impl Trait`, except that it applies to an associated type and not a top-level type alias. 

Example:

```rust
trait RefIterator for Vec<u32> {
    type Item<'me>
    where 
        Self: 'me;

    fn iter<'a>(&'a self) -> impl Iterator<Item = Self:Item<'a>>;
}

// Since 'a is named in the bounds, it is captured.
// `RefIterator` thus becomes:

trait RefIterator for Vec<u32> {
    type Item<'me>
    where 
        Self: 'me;

    type $<'a>: impl Iterator<Item = Self::Item<'a>>
    where 
        Self: 'a; // Implied bound from fn

    fn iter<'a>(&'a self) -> Self::$<'a>;
}
```

## Validity constraint on impls

Given a trait method where `impl Trait` appears in return position,

```rust
trait Trait {
    fn method() -> impl T_0 + ... + T_m;
}
```

where `T_0 + ... + T_m` are bounds, for any impl of that trait to be valid, the following conditions must hold:

* The return type named in the corresponding impl method must implement all bounds `T_0 + ... + T_m` specified in the trait.
* Either the impl method must have `#[refine]`,[^refine] OR
    * The impl must use `impl Trait` syntax to name the corresponding type, and
    * The return type in the trait must implement all bounds `I_0 + ... + I_n` specified in the impl return type. (Taken with the first outer bullet point, we can say that the bounds in the trait and the bounds in the impl imply each other.)

[^refine]: Added in [RFC 3245: Refined trait implementations](https://rust-lang.github.io/rfcs/3245-refined-impls.html).

Additionally, using `-> impl Trait` notation in an impl is only legal if the trait also uses that notation.

```rust
trait NewIntoIterator {
    type Item;
    fn into_iter(self) -> impl Iterator<Item = Self::Item>;
}

// OK:
impl NewIntoIterator for Vec<u32> {
    type Item = u32;
    fn into_iter(self) -> impl Iterator<Item = u32> {
        self.into_iter()
    }
}

// OK:
impl NewIntoIterator for Vec<u32> {
    type Item = u32;
    #[refine]
    fn into_iter(self) -> impl Iterator<Item = u32> + DoubleEndedIterator {
        self.into_iter()
    }
}

// OK:
impl NewIntoIterator for Vec<u32> {
    type Item = u32;
    #[refine]
    fn into_iter(self) -> std::vec::IntoIter<u32> {
        self.into_iter()
    }
}

// Not OK:
impl NewIntoIterator for Vec<u32> {
    type Item = u32;
    fn into_iter(self) -> std::vec::IntoIter<u32> {
        self.into_iter()
    }
}
```

### Interaction with `async fn` in trait

This RFC modifies the “Static async fn in traits” RFC so that async fn in traits may be satisfied by implementations that return `impl Future<Output = ...>` as long as the return-position impl trait type matches the async fn's desugared impl trait with the same rules as above.

```rust
trait Trait {
  async fn async_fn();
  
  async fn async_fn_refined();
}

impl Trait for MyType {
  fn async_fn() -> impl Future<Output = ()> + '_ { .. }
  
  #[refine]
  fn async_fn_refined() -> BoxFuture<'_, ()> { .. }
}
```

## Nested impl traits

Similarly to return-position impl trait in free functions, return position impl trait in traits may be nested in associated types bounds.

Example:

```rust
trait Nested {
    fn deref(&self) -> impl Deref<Target = impl Display> + '_;
}

// This desugars into:

trait Nested {
    type $1<'a>: Deref<Target = Self::$2> + 'a;
    
    type $2: Display;
    
    fn deref(&self) -> Self::$1<'_>;
}
```

But following the same rules as the allowed positions for return-position impl trait, they are not allowed to be nested in trait generics, such as:

``` rust
trait Nested {
    fn deref(&self) -> impl AsRef<impl Sized>; // ❌
}
```

## Dyn safety

To start, traits that use `-> impl Trait` will not be considered dyn safe, *unless the method has a `where Self: Sized` bound*. This is because dyn types currently require that all associated types are named, and the `$` type cannot be named. The other reason is that the value of `impl Trait` is often a type that is unique to a specific impl, so even if the `$` type *could* be named, specifying its value would defeat the purpose of the `dyn` type, since it would effectively identify the dynamic type.

On the other hand, if the method has a `where Self: Sized` bound, the method will not exist on `dyn Trait` and therefore there will be no type to name.

### Dyn safety for `async fn` in trait

This RFC modifies the "Static async fn in traits" RFC to allow traits with `async fn` to be dyn-safe if the method has a `where Self: Sized` bound. This is consistent with how `async fn foo()` desugars to `fn foo() -> impl Future`.

# Drawbacks
[drawbacks]: #drawbacks

This section discusses known drawbacks of the proposal as presently designed and (where applicable) plans for mitigating them in the future.

## Cannot migrate off of impl Trait

In this RFC, if you use `-> impl Trait` in a trait definition, you cannot "migrate away" from that without changing all impls. In other words, we cannot evolve:

```rust
trait NewIntoIterator {
    type Item;
    fn into_iter(self) -> impl Iterator<Item = Self::Item>;
}
```

into 

```rust
trait NewIntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;
    fn into_iter(self) -> Self::IntoIter;
}
```

without breaking semver compatibility for your trait. The [future possibilities](#future-possibilities) section discusses one way to resolve this, by permitting impls to elide the definition of associated types whose values can be inferred from a function return type.

## Clients of the trait cannot name the resulting associated type, limiting extensibility

[As @Gankra highlighted in a comment on this RFC][gankra], the traditional `IntoIterator` trait permits clients of the trait to name the resulting iterator type and apply additional bounds:

[gankra]: https://github.com/rust-lang/rfcs/pull/3193#issuecomment-965505149

```rust
fn is_palindrome<Iter, T>(iterable: Iter) -> bool
where
    Iter: IntoIterator<Item = T>,
    Iter::IntoIter: DoubleEndedIterator,
    T: Eq;
```

The `NewIntoIterator` trait used as an example in this RFC, however, doesn't support this kind of usage, because there is no way for users to name the `IntoIter` type (and, as discussed in the previous section, there is no way for users to migrate to a named associated type, either!). The same problem applies to async functions in traits, which sometimes wish to be able to [add `Send` bounds to the resulting futures](https://rust-lang.github.io/async-fundamentals-initiative/evaluation/challenges/bounding_futures.html).

The [future possibilities](#future-possibilities) section discusses a planned extension to support naming the type returned by an impl trait, which could work to overcome this limitation for clients.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### Does auto trait leakage still occur for `-> impl Trait` in traits?

Yes, so long as the compiler has enough type information to figure out which impl you are using. In other words, given a trait function `SomeTrait::foo`, if you invoke a function `<T as SomeTrait>::foo()` where the self type is some generic parameter `T`, then the compiler doesn't really know what impl is being used, so no auto trait leakage can occur. But if you were to invoke `<u32 as SomeTrait>::foo()`, then the compiler could resolve to a specific impl, and hence a specific [impl trait type alias][tait], and auto trait leakage would occur as normal.

[tait]: https://rust-lang.github.io/impl-trait-initiative/explainer/tait.html

### Can traits migrate from a named associated type to `impl Trait`?

Not compatibly, no, because they would no longer have a named associated type.

### Can traits migrate from `impl Trait` to a named associated type?

Generally yes, but all impls would have to be rewritten.

### Would there be any way to make it possible to migrate from `impl Trait` to a named associated type compatibly?

Potentially! There have been proposals to allow the values of associated types that appear in function return types to be inferred from the function declaration. So the trait has `fn method(&self) -> Self::Iter` and the impl has `fn method(&self) -> impl Iterator`, then the impl would also be inferred to have `type Iter = impl Iterator` (and the return type rewritten to reference it). This may be a good idea, but it is not proposed as part of this RFC.

### What about using a named associated type?

One alternative under consideration was to use a named associated type instead of the anonymous `$` type. The name could be derived by converting "snake case" methods to "camel case", for example. This has the advantage that users of the trait can refer to the return type by name.

We decided against this proposal:

* Introducing a name by converting to camel-case feels surprising and inelegant.
* Return position impl Trait in other kinds of functions doesn't introduce any sort of name for the return type, so it is not analogous.

There is a need to introduce a mechanism for naming the return type for functions that use `-> impl Trait`; we plan to introduce a second RFC addressing this need uniformly across all kinds of functions.

As a backwards compatibility note, named associated types could likely be introduced later, although there is always the possibility of users having introduced associated types with the same name.

### Impl trait in associated type

TODO

# Prior art
[prior-art]: #prior-art

There are a number of crates that do desugaring like this manually or with procedural macros. One notable example is [real-async-trait](https://crates.io/crates/real-async-trait).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- None.

# Future possibilities
[future-possibilities]: #future-possibilities

We expect to introduce a mechanism for naming the result of `-> impl Trait` return types in a follow-up RFC (see the draft [named function types](https://rust-lang.github.io/impl-trait-initiative/RFCs/named-function-types.html) rfc for the current thinking).

Similarly, we expect to be introducing language extensions to address the inability to use `-> impl Trait` types with dynamic dispatch. These mechanisms are needed for async fn as well. A good writeup of the challenges can be found on the "challenges" page of the [async fundamentals initiative](https://rust-lang.github.io/async-fundamentals-initiative/evaluation/challenges/dyn_traits.html).

Finally, it would be possible to introduce a mechanism that allows users to give a name to the associated type that is returned by impl trait. One proposed mechanism is to support an inference mechanism, so that one if you have a function `fn foo() -> Self::Foo` that returns an associated type, the impl only needs to implement the function, and the compiler infers the value of `Foo` from the return type. Another options would be to extend the impl trait syntax generally to let uses give a name to the type alias or parameter that is introduced.
# Microsoft Async Case Study

## Background

Microsoft uses async Rust in several projects both internally and externally. In
this case study, we will focus on how async is used in one project in
particular.

This project manages and interacts with low level hardware resources.
Performance and resource efficiency is key. Async Rust has proven useful not
just because of it enables scalability and efficient use of resources, but also
because features such as cancel-on-drop semantics simplify the interaction
between components.

Due to our constrainted, low-level environment, we use a custom executor but
rely heavily on ecosystem crates to reduce the amount of custom code needed in
our executor.

## Async Trait Usage

The project makes regular use of async traits. Since these are not yet supported
by the language, we have instead used the [`async-trait`] crate.

For the most part this works well, but sometimes the overhead introduced by
boxing the returned fututures is unacceptable. For those cases, we use
[StackFuture], which allows us to emulate a `dyn Future` while storing it in
space provided by the caller.

Now that there is built in support for async in traits in the nightly compiler,
we have tried porting some of our async traits away from the [`async-trait`]
crate.

For many of these the transformation was simple. We merely had to remove the
`#[async_trait]` attribute on the trait and all of its implementations. For
example, we had one trait that looks similar to this:

```rust
#[async_trait]
pub trait BusControl {
    async fn offer(&self) -> Result<()>;
    async fn revoke(&self) -> Result<()>;
}
```

There were several implementations of this trait as well. In this case, all we
needed to do was remove the `#[async_trait]` annotation.

### Send Bounds

In about half the cases, we needed methods to return a future that was `Send`.
This happens by default with `#[async_trait]`, but not when using the built-in
feature.

In these cases, we instead manually desugared the `async fn` definition so we
could add additional bounds. Although these bounds applied at the trait
definition site, and therefore to all implementors, we have not found this to be
a deal breaker in practice.

As an example, one trait that required a manual desugaring looked like this:

```rust
pub trait Component: 'static + Send + Sync {
    async fn save<'a>(
        &'a mut self,
        writer: StateWriter<'a>,
    ) -> Result<(), SaveError>;

    async fn restore<'a>(
        &'a mut self,
        reader: StateReader<'a>,
    ) -> Result<(), RestoreError>;
}
```

The desugared version looked like this:

```rust
pub trait Component: 'static + Send + Sync {
    fn save<'a>(
        &'a mut self,
        writer: StateWriter<'a>,
    ) -> impl Future<Output = Result<(), SaveError>> + Send + 'a;

    fn restore<'a>(
        &'a mut self,
        reader: StateReader<'a>,
    ) -> impl Future<Output = Result<(), RestoreError>> + Send + 'a;
}
```

This also required a change to all the implementation sites since we were
migrating from `async_trait`. This was slightly tedious but basically a
mechanical change.

### Dyn Trait Workaround

We use trait objects in several places to support heterogenous collections of
data that implements a certain trait. Rust nightly does not currently have
built-in support for this, so we needed to find a workaround.

The workaround that we have used so far is to create a `DynTrait` version of
each `Trait` that we need to use as a trait object. One example is the
`Component` trait shown above. For the `Dyn` version, we basically just
duplicate the definition but apply `#[async_trait]` to this one. Then we add a
blanket implementation so that we can triviall get a `DynTrait` for any trait
that has a `Trait` implementation. As an example:

```rust
#[async_trait]
pub trait DynComponent: 'static + Send + Sync {
    async fn save<'a>(
        &'a mut self,
        writer: StateWriter<'a>,
    ) -> Result<(), SaveError>;

    async fn restore<'a>(
        &'a mut self,
        reader: StateReader<'a>,
    ) -> Result<(), RestoreError>;
}

#[async_trait]
impl<T: Component> DynComponent for T {
    async fn save(&mut self, writer: StateWriter<'_>) -> Result<(), SaveError> {
        <Self as Component>::save(self, writer).await
    }

    async fn restore(
        &mut self,
        reader: StateReader<'_>,
    ) -> Result<(), RestoreError> {
        <Self as Component>::restore(self, reader).await
    }
}
```

It is a little annoying to have to duplicate the trait definition and write a
blanket implementation, but there are some mitigating factors. First of all,
this only needs to be done in once per trait and can conveniently be done next
to the non-`Dyn` version of the trait. Outside of that crate or module, the user
just has to remember to use `dyn DynTrait` instead of just `DynTrait`.

The second mitigating factor is that this is a mechanical change that could
easily be automated using a proc macro (although we have not done so).

### Return Type Notation

Since there is an active PR implementing [Return Type Notation][RTN] (RTN), we
gave that a try as well. The most obvious place this was applicable was on the
`Component` trait we have already looked at. This turned out to be a little
tricky. Because `async_trait` did not know how to parse RTN bounds, we had to
forego the use of `#[async_trait]` and use a manually expanded version where
each async function returned `Pin<Box<dyn Future<Output = ...> + Send + '_>>`.
Once we did this, we needed to add `save(..): Send` and `restore(..): Send`
bounds to the places where the `Component` trait was used as a `DynComponent`.
There were only two methods we needed to bound, but this was still mildly
annoying.

The `#[async_trait]` limitation will likely go away almost immediately once the
RTN PR merges, since it can be updated to support the new syntax. Still, this
experience does highlight one thing, which is that the `DynTrait` workaround
requires us to commit up front to whether that trait will guarantee `Send`
futures or not. There does not seem to be an obvious way to push this decision
to the use site like there is with Return Type Notation.

This is something we will want to keep in mind with any macros we create to help
automate these transformations as well as with built in language support for
async functions in trait objects.

[`async-trait`]: https://crates.io/crates/async-trait
[StackFuture]: https://crates.io/crates/stackfuture
[RTN]: https://github.com/rust-lang/rust/pull/109010

## Conclusion

We have not seen any insurmountable problems with async functions in traits as
they are currently implemented in the compiler. That said, it would be
significantly more ergonomic with a few more improvements, such as:

* A way to specify `Send` bounds without manually desugaring a function. Return
  type notation looks like it would work, but even in our limited experience
  with it we have run into ergonomics issues.
* A way to simplify `dyn Trait` support. Language-provided support would be
  ideal, but a macro that automates something like our `DynTrait` workaround
  would be acceptable too.

That said, we are eager to be able to use this feature in production!

# Inline async fn

## Status

Seems unlikely to be adopted, but may be the seed of a better idea

## Status quo

Until now, the only way to make an "async trait" be dyn-safe was to use a manual poll method. The [`AsyncRead`](https://docs.rs/futures/0.3.15/futures/io/trait.AsyncRead.html) trait in futures, for example, is as follows:

```rust
pub trait AsyncRead {
    fn poll_read(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>, 
        buf: &mut [u8]
    ) -> Poll<Result<usize, Error>>;

    unsafe fn initializer(&self) -> Initializer { ... }
    
    fn poll_read_vectored(
        self: Pin<&mut Self>, 
        cx: &mut Context<'_>, 
        bufs: &mut [IoSliceMut<'_>]
    ) -> Poll<Result<usize, Error>> { ... }
}
```

Implementing these traits is a significant hurdle, as it requires the use of `Pin`. It also means that people cannot leverage `.await` syntax or other niceties that they are accustomed to. (See [Alan hates writing a stream](https://rust-lang.github.io/wg-async-foundations/vision/status_quo/alan_hates_writing_a_stream.html) for a narrative description.)

It would be nice if we could rework those traits to use `async fn`. Today that is only possible using the `async_trait` procedural macro:

```rust
#[async_trait]
pub trait AsyncRead {
    async fn read(&mut self, buf: &mut [u8]) -> Result<usize, Error>;

    async fn poll_read_vectored(&mut self, bufs: &mut [IoSliceMut<'_>]) -> Result<usize, Error>;

    unsafe fn initializer(&self) -> Initializer { ... }
    
}
```

Unfortunately, using async-trait has some downsides (see [Alan needs async in traits](https://rust-lang.github.io/wg-async-foundations/vision/status_quo/alan_needs_async_in_traits.html)). Most notably, the traits are rewritten to return a `Box<dyn Future>`. For many purposes, this is fine, but in the case of foundational traits like `AsyncRead`, `AsyncDrop`, `AsyncWrite`, it is a significant hurdle:

* These traits should be included in libcore and available to the no-std ecosystem, like `Read` and `Write`.
* These traits are often on the performance "hot path" and forcing a memory allocation there could be significant for some applications.

There are some other problems with the poll-based design. For example, the buffer supplied to `poll_read` can change in between invocations (and indeed existing adapters take advantage of this sometimes). This means that the traits cannot be used for [zero copy](https://github.com/rust-lang/wg-async-foundations/pull/207), although this is not the only hurdle.

### For drop especially, the state must be embedded within the self type

If we want to have an async version of drop, it is really important that it does not return a separate future, but only makes use of state embedded within the type. This is because we might have a `Box<dyn Future>` or some other type that implements `AsyncDrop`, but where we don't know the concrete type. We are going to want to be able to drop those, which implies that they will live on the stack, which implies that we have to know the contents of the resulting future to know if it is `Send`. 

## Problem: returning a future

The fundamental problem that makes `async fn` not dyn-safe (and the reason that allocation is required) is that every implementation of `AsyncRead` requires different amounts of state. The future that is returned is basically an enumeration with fields for each value that may be live across an `await` point, and naturally that will vary per implementation. This means that code which doesn't know the precise type that it is working with cannot predict how much space that type will require. One solution is certainly boxing, which sidesteps the problem by returning a pointer to memory in the heap.

Using poll methods, as the existing traits do, sidesteps this in a different way: the poll methods basically require that any state that the `AsyncRead` impl requires across invocations of `poll` must be present within the `self` field itself. This is a perfectly valid solution for many applications, but figuring out that state and tracking it efficiently is tedious for users.

## Proposal: "inline" futures

The idea is to allow users to opt-in to "inline futures". Users would write a `repr` attribute on traits that contain `async fn` methods (the attribute):

```rust
#[repr(inline_async)]
trait AsyncRead {
    async fn read(&mut self, buf: &mut [u8]) -> Result<usize, Error>;
    
    ...
}
```

The choice of `repr` is significant here:

* Like repr on a struct, this is meant to be used for things that affect how the code is compiled and its efficiency, but which don't affect the "mental model" of how the trait works.
* Like repr on a struct, using repr may imply some limitations on the things you can do with the trait in order to achieve those benefits.

When a trait is as `repr(inline_async)`, the state for all of its async functions will be added into the type that implements the trait (this attribute could potentially also be used per method). This does imply some key limitations:

* `repr(inline_async)` traits can only be implemented on structs or enums defined in the current crate. This allows the compiler to append those fields into the layout of that struct or enum. 
* `repr(inline_async)` traits can only contain `async fn` with `&mut self` methods.

## Desugaring

The desugaring for an `inline_async` function is different. Rather than an `async fn` becoming a type that returns an `impl Future`, the `async fn` always returns a value of a fixed type. This is a kind of variant on [`Future::PollFn`], which will simply invoke the `poll_read` function each time it is called. What we want is *something* like this, although this doesn't quite work (and relies on unstable features Niko doesn't love):

```rust
trait AsyncRead {
    // Standard async fn desugaring, with a twist:
    fn read(&mut self, buf: &mut [u8]) -> Future::PollFn<
        typeof(<Self as AsyncRead>::poll_read)
    >;

    // 
    fn poll_read(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<usize, Error>>;
}
```

Basically the `read` method would

* initialize the state of the future and then
* construct a [`Future::PollFn`]-like struct that contains a pointer to the `poll_read` function. 

[`Future::PollFn`]: https://github.com/rust-lang/rust/issues/72302

## FAQ

### What's wrong with that desugaring?

The desugaring is pretty close. It has the nice property that, when invoked with a known type, the `Future::PollFn` dispatches statically the poll function, so there is no dynamic dispatch or loss of efficiency.

However, it also has a problem. The return type is still dependent on `Self`, so per our *existing* dyn Rules, that doesn't work. 

It should be possible to extend our dyn Rules, though. All that is needed is a bit of "adaptation glue" in the code that is included in the vtable so that it will convert from a `Future::PollFn` for some fixed `T` to one that uses a `fn` pointer. That seems eminently doable, but I'm not sure if it can be expressed in the language today. 

Pursuing this road might lead to a fundamental extension in dyn safety, which would be nice!

### What state is added precisely to the struct?

* An integer recording the await point where the future is blocked
* Fields for any data that outlives the await

### What if I don't want lots of state added to my struct?

We could limit the use of variables live across an await.

### Could we extend this to other traits?

e.g., simulacrum mentioned `-> impl Iterator` in a (dyn-safe) trait. Seems plausible.

### Why do you only permit `&mut self` methods?

Since the state for the future is stored inline in the struct, we can only have one active future at a time. Using `&mut self` ensures that the poll function is only in use by one future at a time, since that future would be holding an `&mut` reference to the receiver.

### We would like to implement `AsyncRead` for all `&mut impl AsyncRead`, how can we enable that?

I think this *should* be possible. The trick is that the poll function would just dispatch to another poll function. We might be able to support it by detecting the pattern of the `async fn` directly awaiting something reachable from self and supporting that for arbitrary types:

```rust
impl<T: AsyncRead> AsyncRead for &mut T {
    async fn read(&mut self, buf: &mut [u8]) -> Result<usize, Error> {
        T::read(self, buf).await
    }
}
```

Basically this compiles to a `poll_read` that just tweaks dispatches to another `poll_read` with some derefs.

### Can you implement both AsyncRead and AsyncWrite for the same type with this technique?

You can, but you can't simultaneously read and write from the same value. You would need a split-like API. 

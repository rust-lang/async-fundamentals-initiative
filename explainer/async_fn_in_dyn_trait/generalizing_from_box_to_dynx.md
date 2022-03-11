# Generalizing from box with dynx structs

![planning rfc][]

{{#include ../../badges.md}}

So far our design has hardcoded the use of `Box` in the vtable. We can generalize this by introducing a new concept into the compiler; we currently call this a "dynx type"[^better-name]. Like closure types, dynx types are anonymous structs introduced by the compiler. Instead of returning a `Pin<Box<dyn Future>>`, the `next` function will return a `dynx Trait`, which represents "some kind of pointer to a `dyn Trait`". **Note that the `dynx Trait` types are anonymous and that the `dynx` syntax we are using here is for explanatory purposes only.** Users still work with pointer types like `&dyn Trait` , `&mut dyn Trait`, etc.[^user-facing]

[^better-name]: Obviously the name "dynx type" is not great. We were considering "object type" (with e.g. `obj Trait` as the explanatory syntax) but we're not sure what to use here.
[^user-facing]: It may make sense to use `dynx` as the basis for a user-facing feature at some point, but we are not proposing that here.

At runtime, a `dynx Trait` struct has the same size as a `Box<dyn Trait>` (two machine words). If a `dynx Trait` were an ordinary struct, it might look like this:

```rust
struct dynx Trait {
    data: *mut (),
    vtable: *mut (),
}
```

Like a `Box<dyn Trait>`, it carries a vtable that lets us invoke the methods from `Trait` dynamically. Unlike a `Box<dyn Trait>`, it does not hardcode the memory allocation strategy. **Instead, a dynx vtable repurposes the "drop" function slot to mean "drop this pointer and its contents".** This allows `dynx Trait` types to be created from any kind of pointer, such as a `Box`, `&`, `&mut`, `Rc`, or `Arc`. When a `dynx Trait` struct is dropped, it invokes this drop function from its destructor:

```rust
impl Drop for dynx Trait {
    fn drop(&mut self) {
       let drop_fn: fn(*mut ()) = self.vtable.drop_fn;
       drop_fn(self.data);
    }
}
```

## Using dynx in the vtable

Now that we have `dynx`, we can define the vtable for an `AsyncIterator` almost exactly like we saw before, but using `dynx Future` instead of a pinned box:

```rust=
struct AsyncIteratorVtable<I> {
    ..., /* some stuff */
    next: fn(&mut ()) -> dynx Future<Output = Option<I>>
    //                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //                   Look ma, no box!
}
```

Calling `i.next()` for some `i: &mut dyn AsyncIterator` will now give back a `dynx` result:

```rust
i.next().await

// becomes

let f: dynx Future<Output = Option<I>> = i.next();
f.await
```

When the dynx `f` is dropped, its destructor will call the destructor from the vtable, freeing the backing memory in whatever way is appropriate for the kind of pointer that the `dynx` was constructed from.

**What have we achieved?** By using `dynx` in the vtable, we have made it so that the caller doesn't know (or have to know) what memory management strategy is in use for the resulting trait object. It knows that it has an instance of some struct (`dynx Future`, specifically) that implements `Future`, is 2-words in size, and can be dropped in the usual way. 

## The shim function builds the dynx

In the previous section, we saw that using `dynx` in the vtable means that the caller no longer knows (or cares) what memory management strategy is in use. This section explains how the callee actually constructs the `dynx` that gets returned (and how impls can tweak that construction to choose the memory management strategy they want).

Absent any other changes, the `impl` of `AsyncIter` contains an `async fn next()` that returns an `impl Future` which could have any size. Therefore, to construct a `dynx Future`, we are going to need an adaptive shim, just like we used in the `Pin<Box>` example. In fact, the default shim is almost exactly the same as we saw in that case. It allocates a pinned box to store the future and returns it. The main difference is that it converts this `Pin<Box<T>>` into a `dynx Future` before returning, rather than coercing to a `Pin<Box<dyn Future>>` return type (the next section will cover shim functions that don't allocate a box):

```rust
// "pseudocode", you couldn't actually write this because there is no
// user-facing syntax for the `dynx Future` type

fn yielding_range_shim(
    this: &mut YieldingRangeIterator,
) -> dynx Future<Output = Option<u32>> {
    let boxed = Box::pin(<YieldingRangeIterator as AsyncIterator>::next(this));
    
    // invoke the (inherent) `new` method that converts to a `dynx Future`
    <dynx Future>::new(boxed)
}
```

The most interesting part of this function is the last line, which construct the `dynx Future` from its `new` function. Intuitively, the `new` function takes one argument, which must implement the trait `IntoRawPointer` (added by this design). The `IntoRawPointer` trait is implemented for smart pointer types int the standard library, like `Box`, `Pin<Box>`, and `Rc`, and represents "some kind of pointer" as well as "how to drop that pointer":

```rust
// Not pseudocode, will be added to the stdlib and implemented
// by various types, including `Box` and `Pin<Box>`.

unsafe trait IntoRawPointer: Deref {
    /// Convert this pointer into a raw pointer to `Self::Target`. 
    ///
    /// This raw pointer must be valid to dereference until `drop_raw` (below) is invoked;
    /// this trait is unsafe because the impl must ensure that to be true.
    fn into_raw(self) -> *mut Self::Target;
    
    /// Drops the smart pointer itself as well as the contents of the pointer.
    /// For example, when `Self = Box<T>`, this will free the box as well as the
    /// `T` value.
    unsafe fn drop_raw(this: *mut Self::Target);
}
```

The `<dynx Future>::new` method just takes a parameter of type `impl IntoRawPointer` and invokes `into_raw` to convert it into a raw pointer. This raw pointer is then packaged up, together with a modified vtable for `Future`, into the `dynx` structure. The modified vtable is the same as a normal `Future` vtable, except that the "drop" slot is modified so that its drop function points to `IntoRawPointer::drop_raw`, which will be invoked on the data pointer when the `dynx` is dropped. In pseudocode, it looks like this:

```rust
// "pseudocode", you couldn't actually write this because there is no
// user-facing syntax for the `dynx Future` type; but conceptually this
// inherent function exists (in the actual implementation, it may be inlined
// by the compiler, since you could never name it

struct dynx Future<O> {
    data: *mut (),   // underlying data pointer
    vtable: &'static FutureVtable<O>, // "modified" vtable for `Future<Output = O>` for the underlying type
}

struct FutureVtable<O> {
    /// Invokes `Future::poll` on the underlying data.
    ///
    /// Unsafe condition: Expects the output from `IntoRawPointer::into_raw`
    /// which must not have already been freed.
    poll_fn: unsafe fn(*mut (), cx: &mut Context<'_>) -> Ready<O>,
    
    /// Frees the memory for the pointer to future.
    ///
    /// Unsafe condition: Expects the output from `IntoRawPointer::into_raw`
    /// which must not have already been freed.
    drop_fn: unsafe fn(*mut ()),
}

impl<O> dynx Future<Output = O> {
    fn new<RP>(from: RP)
    where
        RP: IntoRawPointer,
        RP::Target: Sized,               // This must be sized so that we know we have a thin pointer.
        RP::Target: Future<Output = O>,  // The target must implement the future trait.
        RP: Unpin,                       // Required because `Future` has a `Pin<&mut Self>` method, see discussion later.
    {
        let data = IntoRawPointer::into_raw(from);
        let vtable = FutureVtable<O> {
            poll_fn: <RP::Target as Future>::poll,
            drop_fn: |ptr| RP::drop_raw(ptr),
        }; // construct vtable
        dynx Future {
            data, vtable
        }
    }
}

impl<O> Future for dynx Future<Output = O> {
    type Output = O;
    
    fn poll(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
    ) -> Ready<()> {
        // Unsafety condition is met since...
        // 1. self.data was initialized in new and is otherwise never changed.
        // 2. drop must not yet have run or else self would not exist.
        unsafe {
            // Conceptually...
            let pin: Pin<RP> = Pin::new(self); // requires `RP: Unpin`.
            let pin_mut_self: Pin<&mut RP::Target> = pin.as_mut();
            self.vtable.poll_fn(pin_mut_self, cx);
            self = pin.into_inner(); // XXX is this quite right?
            
            self.vtable.poll_fn(self.data, cx)
        }
    }
}


impl<O> Drop for dynx Future<Output = O> {
    fn drop(&mut self) {
        // Unsafety condition is met since...
        // 1. self.data was initialized in new and is otherwise never changed.
        // 2. drop must not yet have run or else self would not exist.
        unsafe {
            self.vtable.drop_fn(self.data);
        }
    }
}
```

# How do you create a dynx?


In the previous section, we showed how a `#[dyn(identity)]` function must return "something that can be converted into a dynx struct", and we showed that a case of returning a `Pin<Box<impl Future, A>>` type. But what are the general rules for constructing a `dynx` struct? You're asking the right question, but that's a part of the design we haven't bottomed out yet.

In short, there are two "basic" approaches we could take. One of them is more conservative, in that it doesn't change much about Rust today, but it's also much more complex, because `dyn` dealing with all the "corner cases" of `dyn` is kind of complicated. The other is more radical, but may result in an overall smoother, more coherent design.

Apart from that tantalizing tidbit, we are intentionally not providing the details here, because this document is long enough as it is! The next document dives into this question, along with a related question, which is how `dynx` and sealed traits interact.

This is actually a complex question with (at least) two possible answers.

## Alternative A: P must deref to something that implements Bounds

The pointer type `P` must implement `IntoRawPointer` (along with various other criteria) and its referent must implement `Bounds`.

### IntoRawPointer trait

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

### Other conditions pointer must meet

* Must be `IntoRawPointer` which ensures:
    * `Deref` and `DerefMut` are stable, side-effect free and all that
    * they deref to the same memory as `into_raw`
* If `Bounds` includes a `&mut self` method, `P` must be `DerefMut`
* If `Bounds` includes a `&self` method, `P` must be `Deref`
* If `Bounds` includes `Pin<&mut Self>`, `P` must be `Unpin` ... and ... something something `DerefMut`? how do you get from `Pin<P>` to `Pin<&mut P::Target>`?
* If `Bounds` includes `Pin<&Self>`, `P` must be `Unpin` ... and ... something something `DerefMut`? how do you get from `Pin<P>` to `Pin<&mut P::Target>`?
* If `Bounds` includes an auto trait `AutoTrait`, `P` must implement `AutoTrait`
    * and: `dynx Bounds` implements the auto trait `AutoTrait` (in general, `dynx Bounds` implements all of `Bounds`)
* `Bounds` must be "dyn safe"

## Alternative B: P must implement Bounds

Alternatively, we could declare that the pointer type P must implement `Bounds`. This is much simpler to express, but it has some issues. For example, if you have

```rust
trait Foo { 

}
```

then we could not construct a `dynx Foo` from a `Box<dyn Foo>` because there is no `impl Foo for Box<dyn Foo>`. It would be nice if those impls could be added automatically or at least more easily.

```rust
// Not pseudocode, will be added to the stdlib and implemented
// by various types, including `Box` and `Pin<Box>`.

unsafe trait IntoRawPointer: Deref {
    /// Convert this pointer into a raw pointer to `Self::Target`. 
    ///
    /// This raw pointer must be valid to dereference until `drop_raw` (below) is invoked;
    /// this trait is unsafe because the impl must ensure that to be true.
    fn into_raw(self) -> *mut Self::Target;
    
    /// These methods would be used by compiler to convert back so we can invoke the original
    /// impls.
    unsafe fn from_ref(this: &*mut Self::Target) -> &Self;
    unsafe fn from_mut_ref(this: &mut *mut Self::Target) -> &mut Self;
    ...

    /// Drops the smart pointer itself as well as the contents of the pointer.
    /// For example, when `Self = Box<T>`, this will free the box as well as the
    /// `T` value.
    unsafe fn drop_raw(this: *mut Self::Target);
}
```

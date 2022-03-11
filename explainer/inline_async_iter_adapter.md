# Inline async iter adapter

![planning rfc][]

{{#include ../../badges.md}}

This Appendix demonstrates how the inline async iterator adapter works.

```rust
pub struct InlineAsyncIterator<'me, T: AsyncIterator> {
    iterator: T,
    future: MaybeUninit<T::next<'me>>,
}

impl<T> AsyncIterator for InlineAsyncIterator<T> 
where
    T: AsyncIterator,
{
    type Item = T::Item;

    #[dyn(identity)] // default is "identity"
    fn next<'a>(&'a mut self) -> InlineFuture<'me, T::next<'me>> {
        let future: T::next<'a> = MaybeUninit::new(self.iterator.next());
        // Why do we need this transmute? Is 'a not sufficient?
        let future: T::next<'me> = transmute(future);
        self.future = future;
        InlineFuture::new(self.future.assume_init_mut())
    }
}

pub struct InlineFuture<'me, F> 
where
    F: Future
{
    future: *mut F,
    phantom &'me mut F
}

impl<'me, F> InlineFuture<'me, F> {
    /// Unsafety condition:
    ///
    /// `future` must remain valid for all of `'me`
    pub unsafe fn new(future: *mut F) -> Self {
        Self { future }
    }
}

impl<'me, F> Future for InlineFuture<'me, F> 
where
    F: Future,
{
    fn poll(self: Pin<&mut Self>, context: &mut Context) {
        // Justified by the condition on `new`
        unsafe { ... }
    }
}

impl<F> IntoRawPointer for InlineFuture<F> {
    type Target = F;

    // This return value is the part that has to be thin.
    fn into_raw(self) -> *mut Self::Target {
        self.future
    }

    // This will be the drop function in the vtable.
    unsafe fn drop_raw(this: *mut Self::Target) {
        unsafe { drop_in_place(self.future) }
    }
}

impl<'me> Drop for InlineFuture<'me, F> {
    fn drop(&mut self) {
        unsafe { <Self as IntoRawPointer>::drop_raw(self.future); }
    }
}
```

# Identity shim functions: avoiding the box allocation

![planning rfc][]

{{#include ../../badges.md}}


In the previous section, we explained how the default "shim" created for an `async fn` allocates `Box` to store the future; this `Box` is then converted to a `dynx Future` when it is returned. Using `Box` is a convenient default, but of course it's not always the right choice: for this reason, you can customize what kind of shim using an attribute, `#[dyn]`, attached to the method in the impl:

* `#[dyn(box)]` -- requests the default strategy, allocating a box
* `#[dyn(identity)]` -- requests a shim that just converts the returned future into a `dynx`. The returned future must be of a suitable pointer type (more on that in the next section).

An impl of `AsyncIterator` that uses the default boxing strategy *explicitly* would look like this:

```rust
impl AsyncIterator for YieldingRangeIterator {
    type Item = u32;

    #[dyn(box)]
    async fn next(&mut self) { /* same as above */ }
}
```

If we want to avoid the box, we can instead write an impl for `AsyncIterator` that uses `dyn(identity)`. In this case, the impl is responsible for converting the `impl Future` return value into a an appropriate pointer from which a `dynx` can be constructed. For example, suppose that we are ok with allocating a `Box`, but we want to do it from a custom allocator. What we would like is an adapter `InAllocator<I>` which adapts some `I: AsyncIterator` so that its futures are boxed in a particular allocator. You would use it like this:

```rust
fn example<A: Allocator>(allocator: A) {
    let mut iter = InAllocator::new(allocator, YieldingRangeIterator::new();
    fn_that_takes_dyn(&mut iter);
}

fn fn_that_takes_dyn(x: &mut dyn AsyncIterator) {
    // This call will go into the `InAllocator<YieldingRangeIterator>` and
    // hence will allocate a box using the custom allocator `A`:
    let value = x.next().await;
}
```

To implement `InAllocator<I>`, we first define the struct itself:

```rust
struct InAllocator<A: Allocator, I: AsyncIterator> {
    allocator: A,
    iterator: I,
}

impl<A: Allocator, I: AsyncIterator> InAllocator<A, I> {
    pub fn new(
        allocator: A,
        iterator: I,
    ) -> Self {
        Self { allocator, iterator }
    }
}
```

and then we implement `AsyncIterator` for `InAllocator<..>`, annotating the `next` method with `#[dyn(identity)]`.
The `next` method 

```rust
impl<A, I> AsyncIterator for InAllocator<A, I>
where
    A: Allocator + Clone, 
    I: AsyncIterator,
{
    type Item = u32;

    #[dyn(identity)]
    fn next(&mut self) -> Pin<Box<I::next, A>> {
        let future = self.iterator.next();
        Pin::from(Box::new_in(future, self.allocator.clone()))
    }
}
```

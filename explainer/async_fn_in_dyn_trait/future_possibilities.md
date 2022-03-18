# Future possibilities

![planning rfc][]

{{#include ../../badges.md}}

## Caching across calls

It has been observed that, for many async fn, the function takes a `&mut self` "lock" on the `self` type. In that case, you can only ever have one active future at a time. Even if you are happy to box, you might get a big speedup by caching that box across calls (This is not clear, we'd have to measure, the allocator might do a better job than you can).

We believe that it would be possible to 'upgrade' the ABI to perform this optimization without any change to user code: effectively the compiler would, when calling a function that returns `-> impl Trait` via `dyn`, allocate some "scratch space" on the stack and pass it to that function. The default generated shim, which today just boxes, would make use of this scratch space to stash the box in between calls (this would apply to any `&mut self` function, essentially).

It's also possible to get this behavior today using an adapter.

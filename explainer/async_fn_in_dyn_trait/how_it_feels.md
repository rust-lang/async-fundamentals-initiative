# How it feels to use

![planning rfc][]

{{#include ../../badges.md}}

Let's start with how we expect to *use* `dyn AsyncIterator`. This section will also elaborate some of our desiderata[^showoff], such as the ability to use `dyn AsyncIterator` conveniently in both std and no-std scenarios.

[^showoff]: Ever since I once saw Dave Herman use this bizarre latin plural, I've been in love with it. --nikomatsakis

## How you write a function with a `dyn` argument

We expect people to be able to write functions that take a `dyn AsyncIterator` trait as argument in the usual way:

```rust
async fn count(i: &mut dyn AsyncIterator) -> usize {
    let mut count = 0;
    while let Some(_) = i.next().await {
        count += 1;
    }
    count
}
```

One key part of this is that we want `count` to be invokable from both a std and a no-std environment. 

## How you implement a trait with async fns 

This, too, looks like you would expect.

```rust
struct YieldingRangeIterator {
    start: u32,
    stop: u32,
}

impl AsyncIterator for YieldingRangeIterator {
    type Item = u32;

    async fn next(&mut self) {
        if self.start < self.stop {
            let i = self.start;
            self.start += 1;
            tokio::thread::yield_now().await;
            Some(i)
        } else {
            None
        }
    }
}
```

## How you invoke `count` in std

You invoke it as you normally would, by performing an unsize coercion. Invoking the method requires an allocator by default.

```rust
let x = YieldingRangeIterator::new(...);
let c = count(&mut x /* as &mut dyn AsyncIterator */).await;
```

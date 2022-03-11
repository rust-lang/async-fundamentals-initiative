# Embedded system consuming general purpose libraries

## Summary

* General purpose library defines a trait `Trait` that use async fn
* Embedded library can write a function `consume` that take `&mut dyn Trait` or `&dyn Trait`
* Embedded library can call `consume` without requiring an allocator
    * It does have to jump through some "reasonable" hoops to specify the strategy for allocating the future, and it may not work for all possible traits
    * Can choose from:
        * pre-allocating storage space on the caller stack frame for resulting futures
        * creating an enum that chooses from all possible futures
* The (admittedly vaporware) portability lint can help people discover this

## Status quo

Grace is working on an embedded system. She needs to parse data from an input stream that is formatted as a series of packets in the format TLA. She finds a library `tla` on crates.io with a type that implements the async iterator trait:

```rust
pub struct TlaParser<Source> { ... }

#[async_trait]
impl<Source> AsyncIterator for TlaParser<Source> {
    type Item = TlaPacket;
    async fn next(&mut self) -> Option<Self::Item> {
        ...
    }
}
```

Unfortunately, because `async_trait` desugars to something which uses `Box` internally, she can't use it: she's trying to write for a system with no allocator at all!

*Note:* The *actual* status quo is that the `Stream` trait is not available in std, and the one in the futures crate uses a "poll" method which would be usable by embedded code. But we're looking to a future where we use `async fn` in traits specifically.

## Shiny future

Grace is working on an embedded system. She needs to parse data from an input stream that is formatted as a series of packets in the TLA format. She finds a library `tla` on crates.io with a type that implements the async iterator trait:

```rust
pub struct TlaParser<Source> { ... }

impl<Source> AsyncIterator for TlaParser<Source> {
    type Item = TlaPacket;
    async fn next(&mut self) -> Option<Self::Item> {
        ...
    }
}
```

She has a function that is called from a number of places in the codebase:

```rust
fn example_caller() {
    let mut tla_parser = TlaParser::new(SomeSource);
    process_packets(&mut tla_parser);
}

fn process_packets(parser: &mut impl AsyncIterator<Item = TlaPacket>) {
    while let Some(packet) = parser.next().await {
        process_packet(packet);
    }
}
```

As she is developing, she finds that `process_packets` is being monomorphized many times and it's becoming a significant code size problem for her. She decides to change to `dyn` to avoid that:

```rust
fn process_packets(parser: &mut dyn AsyncIterator<Item = TlaPacket>) {
    while let Some(packet) = parser.next().await {
        process_packet(packet);
    }
}
```

### Tackling portability by preallocating

However, now her code no longer builds! She's getting an error from the portability lint: it seems that invoking `parser.next()` is allocating a box to return the future, and she has specified that she wants to be compatible with "no allocator":

```
warning: converting this type to a `dyn AsyncIterator` requires an allocator
3 |    process_packets(&mut tla_parser);
  |                    ^^^^^^^^^^^^^^^
help: the `dyner` crate offer various a `PreAsyncIterator` wrapper type that can use stack allocation instead
```

Following the recommendations of the portability lint, she investigates the rust-lang `dyner` crate. In there she finds a few adapters she can use to avoid allocating a box. She decides to use the "preallocate" adapter, which preallocates stack space for each of the async functions she might call. To use it, she imports the `PreAsyncIterator` struct (provided by `dyner`) and wraps the `tla_parser` in it. Now she can use `dyn` without a problem:

```rust
use dyner::preallocate::PreAsyncIterator;

fn example_caller() {
    let tla_parser = TlaParser::new(SomeSource);
    let mut tla_parser = PreAsyncIterator::new(tla_parser);
    process_packets(&mut tla_parser);
}

fn process_packets(parser: &mut dyn AsyncIterator<Item = TlaPacket>) {
    while let Some(packet) = parser.next().await {
        process_packet(packet);
    }
}
```

### Preallocated versions of her own traits

As Grace continues working, she finds that she also needs to use `dyn` with a trait of her own devising:

```rust
trait DispatchItem {
    async fn dispatch_item(&mut self) -> Result<(), DispatchError>;
}

struct MyAccumulatingDispatcher { }

impl MyAccumulatingDispatcher {
    fn into_result(self) -> MyAccumulatedResult;
}

fn example_dispatcher() -> String {
    let mut dispatcher = MyAccumulatingDispatcher::new();
    dispatch_things(&mut dispatcher);
    dispatcher.into_result()
}

async fn dispatch_things(context: Context, dispatcher: &mut dyn DispatchItem) {
    for item in context.items() {
        dispatcher.dispatch_item(item).await;
    }
}
```

She uses the `dyner::preallocate::create_struct` macro to create a `PreDispatchItem` struct she can use for dynamic dispatch:

```rust
#[dyner::preallocate::for_trait(PreDispatchItem)]
trait DispatchItem {
    async fn dispatch_item(&mut self) -> Result<(), DispatchError>;
}
```

Now she is able to use the same pattern to call `dispatch_things`. This time she wraps an `&mut dispatcher` instead of taking ownership of `dispatcher`. That works just fine since the trait only has an `&mut self` method. This way she can still call `into_result`:

```rust
fn example_dispatcher() -> MyAccumulatedResult {
    let mut dispatcher = MyDispatcher::new();
    let mut dispatcher = PreDispatchItem::new(&mut dispatcher);
    dispatch_things(&mut dispatcher);
    dispatcher.into_result()
}
```

### Other strategies

Reading the docs, Grace finds a few other strategies available for dynamic dispatch. They all work the same way: a procedural macro generates a custom wrapper type for the trait that handles custom dispatch cases. Some examples:

* Choosing from one of a fixed number of alternatives; returning an enum as the future and not a `Box<impl Future>`.

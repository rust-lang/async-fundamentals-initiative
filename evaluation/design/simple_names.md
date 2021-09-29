# Simple names

One simple way to give names to async functions is to just generate a name based on the method. For example:

```rust
trait AsyncIterator {
    type Item;
    async fn next(&mut self) -> Self::Item;
}
```

could desugar to 

```rust
trait AsyncIterator {
    type Item;
    type Next<'me>: Future<Output = Self::Item> + 'me;
    fn next(&mut self) -> Self::Next<'_>;
}
```

Users could then name the future with `<T as AsyncIterator>::Next<'a>`.

This is a simple solution, but not a very general one, and perhaps a bit surprising (for example, there is no explicit declaration of `Next`).


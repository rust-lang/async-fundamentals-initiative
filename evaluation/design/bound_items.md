# Bound Items

## Summary

Targets the [bounding futures](../challenges/bounding-futures.md) challenge.

This is a series of smaller changes (see detailed design), with the goal of allowing the end user to name the bound described in the challenge text in this way (i'll let the code speak for itself):
```rust
pub trait AsyncIterator {
    type Item;
    async fn next(&mut self) -> Option<Self::Item>;
}

pub async fn start_writing_logs<F>(
    logs: F
) where IsThreadSafeIterator<F>, F: 'static {  // IsThreadSafeIterator is defined below
    todo!()
}

pub bound IsThreadSafeAsyncIterator<T> {
    T: AsyncIterator + Send,
    // Add a `fn` keyword here to refer to the type of associated function.
    <T as AsyncIterator>::fn next: SimpleFnOnceWithSendOutput,
    // Use `*` to refer to potentially long list of all associated functions.
    // this is useful in certain cases.
    <T as AsyncIterator>::fn *: SimpleFnOnceWithSendOutput,
}

```

## Detailed design

I'm not good at naming things, and all names and identifiers are subject to bikeshedding.

* Bound Item(Language construct). Allow user to name a combination of bounds. No `Self` allowed here.
  Could replace the `trait_alias` language feature. Improve ergonomics of many existing code if used
  properly.

  I'm hesitant on which of `PascalCase`, `snake_case`, or `UPPER_CASE` should be used for this naming though.

* Syntax for refering to the type of a associated item in path. Currently using `fn` keyword. Can expand to
  `const` and `type` keywords if needed. Turbofish might be involved if the assoc item is generic.

* `SimpleFnOnce` trait (Language item). A function can only accept one set of arguments,
  so all functions and closures shall implement this, while user-defined callable types doesn't
  have to. This is used for reasoning about the output parameters of associated functions.
  ```rust
  pub trait SimpleFnOnce: FnOnce<Self::Arg> {
      type Arg;
  }
  ```
* `SimpleFnOnceWithSendOutput` trait (Lib construct or user-defined).
  ```rust
  pub trait SimpleFnOnceWithSendOutput : SimpleFnOnce where
      Self::Output: Send
  {}

  impl<T> SimpleFnOnceWithSendOutput for T where T:SimpleFnOnce, T::Output: Send {}
  ```

## What's great about this
* Absolutely minimal type system changes.
* Quite easy to learn. (Name it and use the name)
* Easy to write library documentation, and give examples.

## What's not so great
* New syntax item constructs. Three parsers (rustc, ra, syn) needs to change to support this.
* Very verbose code writing style. Many more lines of code.
* Will become part of library API in many cases.

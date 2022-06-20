# Async fn in traits usage scenarios

What follows are a list of "usage scenarios" for async fns in traits and some of their characteristics.

## Core Rust async traits

Defining core Rust async traits like `AsyncRead`:

```rust
trait AsyncRead {
    async fn read(&mut self, buffer: &mut [u8]) -> usize;
}
```

And implementing them in client code:

```rust
```

Characteristics:

* Need static dispatch with zero overhead, so folks can write `fn foo<F: AsyncRead>()`, have it fully monomorphized, and equally efficient
* Need to support `dyn AsyncRead` eventually (people will want to do that)
* Need ability to 

## Client: ship library in crates.io that leverages async traits

Example from AWS Rust SDK:

```rust
pub trait ProvideCredentials: Send + Sync + Debug {
    fn provide_credentials<'a>(&'a self) -> ProvideCredentials<'a>;
}
```

People implement this like

```rust
impl ProvideCredentials for MyType {
    fn provide_credentials<'a>(&'a self) -> ProvideCredentials<'a> {
        ProvideCredentials::new(async move { ... })
    }
}
```

where [`ProvideCredentials](https://docs.rs/aws-types/0.10.1/aws_types/credentials/future/struct.ProvideCredentials.html) is a struct that takes an `impl Future` and creates

## Embedded: ship library in crates.io that leverages async traits

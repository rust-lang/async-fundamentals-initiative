# Taking ownership of the receiver

## Summary

* Support traits that use `fn(self)` with dynamic dispatch
    * The caller will have to be using a `Box<dyn Foo>`, but that is not hard-coded into the trait.

## Status quo

Grace is working on an embedded system. She needs to parse data from an input stream that is formatted as a series of packets in the format TLA. She finds a library `tla` on crates.io with a type that implements the async iterator trait:

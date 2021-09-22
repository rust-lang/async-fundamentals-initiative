# Guaranteeing async drop

One challenge with `AsyncDrop` is that we have no guarantee that it will be used. For any type `MyStruct` that implements `AsyncDrop`, it is always possible in Rust today to drop an instance of `MyStruct` in synchronous code. In that case, we cannot run the async drop. What should we do?

Obvious alternatives:

* Panic or abort
* Use some form of "block on" or other default executor to execute the asynchronous await
* Extend Rust in some way to prevent this condition.

We can also mitigate this danger through lints (e.g., dropping value which implements AsyncDrop).

Some types may implement both synchronous and asynchronous drop.
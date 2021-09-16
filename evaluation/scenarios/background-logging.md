# Background logging

In this scenario, the `start_writing_logs` function takes an async iterable and spawns out a new task. This task will pull items from the iterator and send them to some server:

```rust
trait AsyncIterator {
    type Item;
    async fn next(&mut self) -> Self::Item;
}

// Starts a task that will write out the logs in the background
async fn start_writing_logs(
    logs: impl AsyncIterator<Item = String> + 'static
) {
    spawn(async move || {
        while let Some(log) = logs.next().await {
            send_to_serve(log).await;
        }
    });
}
```

The precise signature and requirements for the `spawn` function here will depend on what [kind of executor](../executor-styles.md) you are using, so let's consider each case but let's consider each case separately.

One note: in [tokio] and other existing executors, the `spawn` function takes a future, not an async closure. We are using a closure here because that is more analogous to the synchronous signature, but also because it enables a distinction between the *initial state* and the future that runs.

## Thread-local executor

This is the easy case. Nothing has to be `Send`.

## Work-stealing executor

In this case, the spawn function will require both that the initial closure itself is `Send` and that the future it returns is `Send` (so that it can be moved from place to place as code executes).

We don't have a good way to express this today! The problem is that there is a future that results from calling `logs.next()`, let's call it `F`. The future to be *spawned* has to be sure that `F: Send`. There isn't a good way to do this today, and even explaining the problem is surprisingly hard. Here is a "desugared version" of the program that shows what is needed:

```rust
trait AsyncIterator {
    type Item;
    type NextFuture: Future<Output = Self::Item>;

    fn next(&mut self) -> impl Self::NextFuture;
}

// Starts a task that will write out the logs in the background
async fn start_writing_logs<I>(
    logs: I
) 
where
    I: AsyncIterator<Item = String> + 'static + Send,
    I::NextFuture: Send,
{
    spawn(async move || {
        while let Some(log) = logs.next().await {
            send_to_serve(log).await;
        }
    });
}
```

(With [RFC 2289], you could write `logs: impl AsyncIterator<Item = String, NextFuture: Send> + Send`, which is more compact, but still awkward.)

[RFC 2289]: https://rust-lang.github.io/rfcs/2289-associated-type-bounds.html
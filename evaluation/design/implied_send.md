# Implied Send

## Status

❌ Rejected. This idea can be quite [productive](https://rustacean-principles.netlify.app/how_rust_empowers/productive.html), but it is not [versatile](https://rustacean-principles.netlify.app/how_rust_empowers/versatile.html) (it rules out important use cases) and it is not [supportive](https://rustacean-principles.netlify.app/how_rust_empowers/supportive.html) (it is confusing).

(FIXME: I think the principles aren't quite capturing the constriants here! We should adjust.)

## Summary

Targets the [bounding futures](../challenges/bounding-futures.md) challenge.

The core idea of **"implied Send"** is to say that, by default at least, the future that results from an `async fn` must be `Send` if the `Self` type that implements the trait is `Send`.

In Chalk terms, you can think of this as a bound like 

```
if (Implemented(Self: Send)) { 
    Implemented(Future: Send)
}
```

Mathematically, this can be read as `Implemented(Self: Send) => Implemented(Future: Send)`. In other words, if you assume that `Self: Send`, then you can show that `Future: Send`.

## Desugared semantics

If we extended the language with `if` bounds a la Chalk, then the desugared semantics of "implied send" would be something like this:

```rust
trait AsyncIterator {
    type Item;
    type NextFuture: Future<Output = Self::Item>
                   + if (Self: Send) { Send };

    fn next(&mut self) -> impl Self::NextFuture;
}
```

As a result, when you implement `AsyncIterator`, the compiler will check that your futures are `Send` if your input type is assumed to be `Send`.

## What's great about this

The cool thing about this is that if you have a bound like `T: AsyncIterator + Send`, that automatically implies that any futures that may result from calling `AsyncIterator` methods will also be `Send`. Therefore, the [background logging](../scenarios/background-logging.md) scenario works like this, which is perfect for a [work stealing](../executor-styles.md) executor style.

```rust
async fn start_writing_logs(
    logs: impl AsyncIterator<Item = String> + Send + 'static
) {
    ...
}
```

## What's not so great

### Negative reasoning: Semver interactions, how to prove

In this proposal, when one implements an async trait for some concrete type, the compiler would presumably have to first check whether that type implements `Send`. If not, then it is ok if your futures do not implement `Send`. That kind of negative reasoning is actually quite tricky -- it has potential semver implications, for example -- although auto traits are more amenable to it than other things, since they already interact with semver in complex ways.

In fact, if we use the standard approach for proving implication goals, the setup would not work at all. The typical approach to proving an implication goal like `P => Q` is to assume `P` is true and then try to prove `Q`. But that would mean that we would just wind up assuming that the `Self` type is `Send` and trying to use that to prove the resulting `Future` is `Send`, not *checking* whether `Self` is `Send` to decide.

### Not analogous to async fn outside of traits

With inherent async functions, we don't check whether the resulting future is `Send` right away. Instead, we remember what state it has access to, and then if there is some part of the code that requires a future to be `Send`, we check *then*. 

But this "implied send" approach is different: the trait is effectively declaring up front that async functions must be send (if the Self is send, at least), and so you wind up with errors at the *impl*. This is true regardless of whether the future ever winds up being used in a spawn.

The concern here is not *precisely* that the result is too strict (that's covered in the next bullet), but rather that it will be surprising behavior for people. They'll have a hard time understanding why they get errors about `Send` in some cases but not others.

### Stricter than is required for non-work-stealing [executor styles]

Building on the previous point, this approach can be stricter than what is required when not using a work stealing [executor style].

As an example, consider a case where you are coding in a thread-local setting, and you have a struct like the following

```rust
struct MyCustomIterator {
    start_index: u32
}
```

Now you try to implement `AsyncIterator`. You know your code is thread-local, so you decide to use some `Rc` data in the process:

```rust
impl AsyncIterator for MyCustomIterator {
    async fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        let ssh_key = Rc::new(vec![....]);
        do_some_stuff(ssh_key.clone());
        something_else(self.start_index).await;
    }
}
```

But now you get a compilation error:

```
error: `read` must be `Send`, since `MyCustomIterator` is `Send`
```

Frustrating!

[executor styles]: ../executor-styles.md
[executor style]: ../executor-styles.md

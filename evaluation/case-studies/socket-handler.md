# Netstack3 Async Socket Handler Case Study

This case study presents a simplification of a common handler pattern enabled by `async_fn_in_trait` (AFIT).

## Background

[Netstack3] is a networking stack written in Rust for the [Fuchsia operating
system][Fuchsia]. As a Fuchsia component, Netstack3 communicates with its
clients using Fuchsia-native asynchronous IPC via generated bindings from
Fuchsia interface definition language ([FIDL]) specifications. Netstack3
provides, via various FIDL protocols, the ability for other Fuchsia components
(e.g. applications) to create and manipulate POSIX-like socket objects.

## Per-socket handler implementations

To allocate a socket, a Fuchsia component sends a single message to the netstack
indicating the desired type of socket, along with a [Zircon channel] that the
netstack is expected to listen for requests on. The other end of the channel is
held by the client and used to send requests for the new socket to the netstack.

When Netstack3 receives a request to create a socket, it spawns a new
[`fuchsia_async::Task`] to dispatch incoming requests for the socket to the
socket's handler. The type of messages for the socket depends on the protocol,
so Netstack3 has distinct handler types for each.

Though the handler types are distinct, their functionality is fairly similar:

- Handlers wait to receive incoming requests on their channel, then process
  them.
- Handlers process each received message completely before polling for the next.
- Handlers support `Clone` requests by merging a new stream of incoming
  requests with their existing one.
- Handlers exit when either their request stream ends or in response to an
  explicit `Close` message.
- On exit, handlers clean up any resources corresponding to the socket.

Request handling for a given socket is done using `async`/`await`. Though each
individual socket handles requests serially, this allows requests for different
sockets to be handled concurrently by the executor.

## Individual implementations

Before the introduction of `async_fn_in_trait`, Netstack3's socket handlers were
each implemented independently, with minimal code reuse. This resulted in
significant duplication of code for the common behaviors above. The
straightforward refactor would define a generic `SocketWorker<H>` type with
the shared behavior, and that would delegate to an implementer of a trait
`SocketHandler` for socket-type-specific request handling:

```rust
pub struct SocketWorker<H> {
    handler: H,
}

pub trait SocketHandler: Default {
    type Request;

    /// Handles a single request.
    async fn handle_request(
        &mut self,
        request: Self::Request,
    ) -> ControlFlow<(), Option<RequestStream<Self::Request>>;

    /// Closes the socket managed by this handler.
    fn close(self);
}

impl<H: SocketHandler> SocketWorker<H> {
    /// Starts servicing events from the provided event stream.
    pub async fn serve_stream(stream: RequestStream<H>) {
        Self { handler: H::default() }.handle_stream(stream)
    }

    async fn handle_stream(mut self, requests: RequestStream<H>) {
      let Self {handler} = self;
      while let Some(request) = requests.next().await {
        // Call `handler.handle_request()` for each request while merging
        // new request streams into `requests`.
      }
      handler.close()
    }
}
```

Because `SocketHandler::handle_request` is an `async fn`, this won't compile on
the current version of stable Rust (1.68.0 as of writing). There are a
[couple options][AFIT workarounds] for working around the lack of support for
AFIT, but they each have significant downsides:

### Option 1: Make request handling a hand-rolled `Future` impl on a custom type

One way to work around the lack of AFIT is to declare a non-`async` trait
function that returns an instance of an associated type that implements
`Future`. This works, but requires explicitly structuring control flow as state
within the associated type. For Netstack3, where request handling branches down
tens of paths, maintaining this state machine by hand would be impractically
difficult.

### Option 2: Use dynamic dispatch

This is similar to the above, but instead of an associated type, the non-`async`
function returns a `Box<dyn Future>`. The trait implementation can then call an
`async fn`, then box up and return the result. This results in more readable
code than option 1 at the cost of allocation and dynamic dispatch at run time.
For Netstack3, which is on the critical path of every network-connected
component in the system, this is not worth the benefits of abstraction.

## With AFIT

Using `async_fn_in_trait` allowed performing the refactoring proposed above
without workarounds. The resulting code doesn't use dynamic dispatch, and the
implementations of `SocketHandler::handle_request` are written as
regular `async fn`s. The full definition of the abstract worker and handler
trait can be found in the [Netstack3 source code][socket worker source]).

## `Send` bound limitation

One of the current limitations for AFIT is the
[inability to specify bounds][AFIT spawn from generic] on the type of the future
returned from an async trait function. This can cause errors when the caller of
the function requires a `Send` bound so that the future can be passed to a
multi-threaded executor.

Netstack3 uses [`fuchsia_async::Task::spawn`] to create tasks that can run on
Fuchsia's multi-threaded executor, and so initial attempts to use AFIT ran afoul
of the limitation. Luckily, the suggested workaround of moving the spawn point
out of generic code worked for Netstack3: `Task::spawn` is [called in
socket-specific code][datagram diff] instead of within generic socket worker
code. Since the compiler has access to the concrete `Future`-implementing type
returned by the specific impl of `SocketHandler::handle_request`, it can verify
that it and all its callers implement `Send`.

## Future usages

While AFIT is currently being used in Netstack3 for abstracting over socket
behaviors, it's likely that there are other places where it would prove useful,
including in some of the existing per-IP-version code with common logic.

[Netstack3]: https://fuchsia.dev/fuchsia-src/contribute/roadmap/2021/netstack3
[Fuchsia]: https://fuchsia.dev/
[FIDL]: https://fuchsia.dev/fuchsia-src/development/languages/fidl
[Zircon channel]: https://fuchsia.dev/fuchsia-src/reference/kernel_objects/channel?hl=en
[`fuchsia_async::Task`]: https://fuchsia-docs.firebaseapp.com/rust/fuchsia_async/struct.Task.html
[AFIT workarounds]: https://blog.rust-lang.org/inside-rust/2022/11/17/async-fn-in-trait-nightly.html#workarounds-available-in-the-stable-compiler
[socket worker source]: https://cs.opensource.google/fuchsia/fuchsia/+/d38d260043834b14b5dbf8e315ef11c712f12166:src/connectivity/network/netstack3/src/bindings/socket/worker.rs
[AFIT spawn from generic]: https://blog.rust-lang.org/inside-rust/2022/11/17/async-fn-in-trait-nightly.html#limitation-spawning-from-generics
[`fuchsia_async::Task::spawn`]: https://fuchsia-docs.firebaseapp.com/rust/fuchsia_async/struct.Task.html#method.spawn
[datagram diff]: https://fuchsia.googlesource.com/fuchsia/+/95808a53ec3507aeaa2b185363b45b72553e1d5e%5E%21/#F1
# Introduction

Modrpc is a RPC (remote procedure call) framework much like gRPC, Thrift, and Cap'n Proto. Users describe their application's interface in modrpc's interface definition language (IDL), and boilerplate code to implement the interface's participants is automatically generated in any of the languages that modrpc supports. Here's what a simple modrpc schema looks like:

```
import std ".modrpc/std.modrpc"

interface Chat @(Client, Server) {
    objects {
        register: std.Request<
            RegisterRequest,
            result<void, RegisterError>,
        > @(Client, Server),

        send_message: std.Request<
            SendMessageRequest,
            result<void, SendMessageError>,
        > @(Client, Server),
    }

    state {
        users: [RegisteredUser],
    }
}

struct RegisteredUser {
    endpoint_addr: u64,
    alias: string,
}

struct RegisterRequest {
    alias: string,
}

enum RegisterError {
    UserAlreadyExists,
    Internal,
}

struct SendMessageRequest {
    content: string,
}

enum SendMessageError {
    Internal,
    InsufficientRizz,
}
```

Check out [chat-modrpc-example](https://github.com/modrpc-org/chat-modrpc-example) to see a working example application.

Modrpc focuses on the following areas:
- modularity
- portability
- RPC over multicast
- performance

## Portability

This refers to making modrpc usable in as many situations as possible. There are a few design goals aimed at that (some are currently aspirational):
- Don't allocate after startup (important for embedded use-cases)
- Be transport-agnostic - support communication over shared-memory, TCP, WebSockets, or even an embedded radio.
- Try to be as lightweight as possible.

## RPC over multicast

Here "multicast" means events for a modrpc interface being broadcasted to all interested peers. This allows you to, for example, have all clients see each others' requests and the server's responses to those requests. This is useful when developing collaborative apps and in many cases allows you to avoid building dedicated mechanisms to sync state among peers.

## Performance

Modrpc aims to be lightweight and easy to setup in simple single-threaded scenarios, but also aims to be easily scalable to support high-throughput, high concurrency scenarios (tens to hundreds of thousands of tasks interacting with the RPC system). The runtime is written in Rust and is `async` and thread-per-core. Some examples of performance-oriented design decisions:

- Avoid frequent heap allocations
    - A fixed-size, async buffer pool is used to allocate messages.
    - Messages can be serialized and deserialized without allocations.
- Batching
    - Threads grab buffers from the shared pool in batches to allocate messages on.
    - Multiple messages can be backed by a single buffer.
    - Written buffers are flushed for sending out on a transport in batches.
    - Inter-thread message queues automatically batch under high load.
- Thread-local waker registrations
    - Use non-`Sync` async datastructures where possible.
    - In some datastructures (for example `modrpc::HeapBufferPool`), when there are many tasks on multiple threads that need to be notified, only one task per thread will register itself in a thread-safe queue - others will wait in a thread-local queue.
- Scaling across cores / load balancing
    - Message handlers on multiple threads can subscribe to a load-balancing mpmc queue to receive work.
    - A message will be processed directly on the received thread if possible to circumvent the mpmc queue and some buffer reference counting.

Check out the [local-benchmark](https://github.com/modrpc-org/modrpc/blob/c8be8fde5eb34fcc23949bd2cc712e1939298b10/examples/local-benchmark/src/main.rs) example application to get a sense for what configuring a multithreaded modrpc runtime with multiple transports looks like.

To get an idea of current performance, try out the [p2p-benchmark](https://github.com/modrpc-org/modrpc/tree/main/examples/p2p-benchmark) example application - it has a single-threaded server being spammed with very cheap requests by a multi-threaded client. On my laptop the server is able to serve 3.8M+ requests/second.

## Modularity

Modrpc allows users to define interfaces that can be implemented once in Rust and imported and re-used as components of larger interfaces. In the earlier example, we encountered the `std.Request` interface from modrpc's standard library. As you might've guessed, this interface provides the request-response pattern that comes baked into traditional RPC frameworks.

The definition of `std.Request`:
```
interface Request<Req, Resp> @(Client, Server) {
    events @(Client) -> @(Client, Server) {
        private request: Request<Req>,
    }

    events @(Server) -> @(Client) {
        private response: Response<Resp>,
    }

    impl @(Server) {
        handler: async Req -> Resp,
    }

    methods @(Client) {
        call: async Req -> Resp,
    }
}
```

All interfaces boil down to events that can be sent from some set of roles to some other set of roles. In the case of `std.Request`, only clients can send requests, and both clients and servers will receive them. This allows clients to observe requests made by other clients when doing RPC-over-multicast.

Common logic for a re-usable interface is hand-written once in Rust, and downstream interfaces and applications can invoke the logic via methods that the re-usable interface exposes. An example of this is the `call` method on `std.Request`.

If the re-usable interface requires application-specific logic, these are specified via `impl` blocks. In the case of `std.Request`, implementers of a request server must provide an async function that receives a request and produces a response.

Under the hood, the re-usable Rust implementation of the `Request` interface handles request payload encapsulation and response tracking / notification at the [client](https://github.com/modrpc-org/modrpc/blob/c8be8fde5eb34fcc23949bd2cc712e1939298b10/std-modrpc/rust/src/role_impls/request_client.rs#L192-L248) and calling the user's supplied request handler at the [server](https://github.com/modrpc-org/modrpc/blob/c8be8fde5eb34fcc23949bd2cc712e1939298b10/std-modrpc/rust/src/role_impls/request_server.rs#L91-L115).

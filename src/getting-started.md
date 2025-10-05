# Getting Started

This guide will walk you through starting your first modrpc project. It assumes you have already installed Rust - see [rustup.rs](https://rustup.rs).

### 1. Install the modrpc tooling

```
cargo install modrpcc
```

### 2. Create the structure of your modrpc project

```
mkdir my-modrpc-app
cd my-modrpc-app

cargo new --bin --name my-server server
cargo new --bin --name my-client client
```

### 3. Download the modrpc standard library

Currently the modrpc standard library is unversioned. Download the latest proof-of-concept schema straight from the GitHub repo - it will be kept in-sync with the latest `std-modrpc` on crates.io.

```
mkdir .modrpc
curl https://raw.githubusercontent.com/modrpc-org/modrpc/refs/heads/main/proto/std.modrpc -o .modrpc/std.modrpc
```

### 4. Define your first interface

Populate `my-app.modrpc`:
```
import std ".modrpc/std.modrpc"

interface MyApp @(Client, Server) {
    objects {
        compute_fizzbuzz: std.Request<
            ComputeFizzbuzzRequest,
            result<ComputeFizzbuzzSuccess, ComputeFizzbuzzError>,
        > @(Client, Server),
    }
}

struct ComputeFizzbuzzRequest {
    i: u64,
}

struct ComputeFizzbuzzSuccess {
    message: string,
}

enum ComputeFizzbuzzError {
    InvalidRequest,
}
```

### 5. Generate the modrpc glue library for your interface

This will generate a Rust crate at `my-app-modrpc/rust`:
```
modrpcc --language rust --output-dir . --name my-app my-app.modrpc
```

### 6. Implement your server

Move to the server's directory:
```
cd server
```

Populate `Cargo.toml`:
```toml
[package]
name = "my-server"
version = "0.1.0"
edition = "2024"

[dependencies]
modrpc = { version = "0.0", features = ["tcp-transport"] }
modrpc-executor = { version = "0.0", features = ["tokio"] }
my-app-modrpc = { path = "../my-app-modrpc/rust" }
std-modrpc = "0.0"
tokio = "1"
```

Populate `src/main.rs`:
```rust
use modrpc_executor::ModrpcExecutor;

fn main() {
    let mut ex = modrpc_executor::TokioExecutor::new();
    let _guard = ex.tokio_runtime().enter();

    let buffer_pool = modrpc::HeapBufferPool::new(256, 4, 4);
    let rt = modrpc::RuntimeBuilder::new_with_local(ex.spawner());
    let (rt, _rt_shutdown) = rt.start::<modrpc_executor::TokioExecutor>();

    ex.run_until(async move {
        let tcp_server = modrpc::TcpServer::new();
        let listener = tokio::net::TcpListener::bind("0.0.0.0:9090").await
            .expect("tcp listener");

        loop {
            println!("Waiting for client...");
            let (stream, client_addr) = match listener.accept().await {
                Ok(s) => s,
                Err(e) => {
                    println!("Failed to accept client: {}", e);
                    continue;
                }
            };
            stream.set_nodelay(true).unwrap();

            let _ = tcp_server.accept_local::<my_app_modrpc::MyAppServerRole>(
                &rt,
                buffer_pool.clone(),
                stream,
                start_my_app_server,
                my_app_modrpc::MyAppServerConfig { },
                my_app_modrpc::MyAppInitState { },
            )
            .await
            .unwrap();

            println!("Accepted client {}", client_addr);
        }
    });
}

fn start_my_app_server(
    cx: modrpc::RoleWorkerContext<my_app_modrpc::MyAppServerRole>,
) {
    cx.stubs.compute_fizzbuzz.build(cx.setup, async move |_source, request| {
        use my_app_modrpc::{ComputeFizzbuzzError, ComputeFizzbuzzSuccess};

        let Ok(i) = request.i() else {
            return Err(ComputeFizzbuzzError::InvalidRequest);
        };

        println!("Received request: {i}");

        let response = if i % 3 == 0 && i % 5 == 0 {
            ComputeFizzbuzzSuccess { message: "FizzBuzz".to_string() }
        } else if i % 3 == 0 {
            ComputeFizzbuzzSuccess { message: "Fizz".to_string() }
        } else if i % 5 == 0 {
            ComputeFizzbuzzSuccess { message: "Buzz".to_string() }
        } else {
            ComputeFizzbuzzSuccess { message: format!("{i}") }
        };

        println!("  response: {response:?}");

        Ok(response)
    });
}
```

Build and run the server:
```
cargo run --release
```

### 7. Implement your client

In another terminal session, move to the client's directory:
```
cd /path/to/my-modrpc-app/client
```

Populate `Cargo.toml`:
```toml
[package]
name = "my-client"
version = "0.1.0"
edition = "2024"

[dependencies]
modrpc = { version = "0.0", features = ["tcp-transport"] }
modrpc-executor = { version = "0.0", features = ["tokio"] }
my-app-modrpc = { path = "../my-app-modrpc/rust" }
std-modrpc = "0.0"
tokio = "1"
```

Populate `src/main.rs`:
```rust
use modrpc_executor::ModrpcExecutor;

fn main() {
    let mut ex = modrpc_executor::TokioExecutor::new();
    let _guard = ex.tokio_runtime().enter();

    let buffer_pool = modrpc::HeapBufferPool::new(256, 4, 4);
    let rt = modrpc::RuntimeBuilder::new_with_local(ex.spawner());
    let (rt, _rt_shutdown) = rt.start::<modrpc_executor::TokioExecutor>();

    ex.run_until(async move {
        let stream = tokio::net::TcpStream::connect("127.0.0.1:9090").await
            .expect("tcp stream connect");
        stream.set_nodelay(true).unwrap();

        println!("Connected to server");

        let connection =
            modrpc::tcp_connect::<my_app_modrpc::MyAppClientRole>(
                &rt,
                buffer_pool,
                modrpc::WorkerId::local(),
                my_app_modrpc::MyAppClientConfig { },
                stream,
            )
            .await
            .unwrap();
        let my_app_client = connection.role_handle;

        for i in 1..=15 {
            let response = my_app_client.compute_fizzbuzz.call(
                my_app_modrpc::ComputeFizzbuzzRequest { i }
            )
            .await
            .expect("fizzbuzz failed");

            println!("{}", response.message);
        }
    });
}
```

Build and run the client:
```
cargo run --release
```

The end!

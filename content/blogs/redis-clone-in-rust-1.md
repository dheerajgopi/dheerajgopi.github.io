---
title: "Redis Clone in Rust - Part 1"
subtitle: "Setting Up a Simple TCP Server"
date: 2024-08-24T00:00:00+05:30
draft: true
tags: ["rust", "redis", "tokio", "tcp"]
---

In this tutorial, you'll start the journey towards building [Nimblecache](https://github.com/dheerajgopi/nimblecache) (Redis clone) by first creating a simple TCP server using [Tokio](https://tokio.rs/), a powerful asynchronous runtime for the Rust programming language. By the end of this tutorial, you will have a basic understanding of how to handle TCP connections and manage asynchronous tasks using Tokio. Let's dive in!

## Why Tokio?

We could use standard threads to build the TCP server, as it is simpler than using something like Tokio. However, we are choosing Tokio because its async model is perfect for TCP servers. It handles concurrent connections efficiently and uses fewer system resources than a threaded model. It excels in I/O efficiency by allowing the server to perform other tasks while waiting for network operations, unlike standard threads, which require a separate thread for each connection and can quickly exhaust resources.

## What we're building

Before we dive into the code, let's briefly cover what we're building:

* A TCP server that listens for incoming connections
* Handles multiple clients concurrently
* Responds with a simple "Hello!" message to each client


This TCP server will serve as a foundation on which we can build the Redis clone.

## Setting Up the Project

First, create a new Rust project:

```bash
cargo new nimblecache
cd nimblecache
```

Add the following dependencies to your `Cargo.toml`:

```ini
[package]
name = "nimblecache"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
anyhow = "1.0.86"
tokio = { version = "1.38.0", features = [
    "rt-multi-thread",
    "macros",
    "net",
    "io-util",
    "sync",
] }
log = "0.4.22"
env_logger = "0.11.3"
```

[anyhow](https://docs.rs/anyhow/latest/anyhow/): Anyhow is an error-handling library. It simplifies error management by providing `anyhow::Error` and `anyhow::Result` types. These can be used to handle various error types without excessive boilerplate.

[tokio](https://docs.rs/tokio/latest/tokio/): Tokio is the asynchronous runtime for Rust. The features enabled are:

* `"rt-multi-thread"`: Enables the multi-threaded runtime.
* `"macros"`: Includes helpful macros like `#[tokio::main]`.
* `"net"`: Provides networking types like `TcpListener` and `TcpStream`.
* `"io-util"`: Includes I/O utilities for working with streams.
* `"sync"`: Offers synchronization primitives (like tokio `Mutex`) for concurrent programming.


The Tokio dependency is more specific in its feature selection, enabling only the necessary features instead of using `features = ["full"]`. This approach helps achieve faster compile times and a smaller binary size.

[log](https://docs.rs/log/latest/log/): This is the logging facade for Rust. It provides macros for logging at different levels (`info`, `warn`, `error` etc.).

[env\_logger](https://docs.rs/env_logger/latest/env_logger/): This is a logger implementation that can be configured via environment variables. It works in conjunction with the `log` crate.

## The server

The code below is the core of the TCP server. It defines a `Server` struct, which encapsulates the main functionality of our TCP server:

1. It holds a `TcpListener` which listens for incoming connections.
2. The `run` method enters an infinite loop, accepting new connections.
3. For each connection, it spawns a new asynchronous task to handle the client.
4. The `accept_conn` method wraps the process of accepting a new connection.


It's a basic implementation with detailed comments to make it easy to understand.

```rust
// src/server.rs

// anyhow provides the Error and Result types for convenient error handling
use anyhow::{Error, Result};

// log crate provides macros for logging at various levels (error, warn, info, debug, trace)
use log::error;

use tokio::{
    // AsyncWriteExt trait provides asynchronous write methods like write_all
    io::AsyncWriteExt,
    net::{TcpListener, TcpStream},
};

/// The Server struct holds the tokio TcpListener which listens for
/// incoming TCP connections.
#[derive(Debug)]
pub struct Server {
    listener: TcpListener,
}

impl Server {
    /// Creates a new Server instance with the given TcpListener.
    pub fn new(listener: TcpListener) -> Server {
        Server { listener }
    }

    /// Runs the server in an infinite loop, continuously accepting and handling
    /// incoming connections.
    pub async fn run(&mut self) -> Result<()> {
        loop {
            // accept a new TCP connection.
            // If successful the corresponding TcpStream is stored
            // in the variable `sock`, else a panic will occur.
            let mut sock = match self.accept_conn().await {
                Ok(stream) => stream,
                // Log the error and panic if there is an issue accepting a connection.
                Err(e) => {
                    error!("{}", e);
                    panic!("Error accepting connection");
                }
            };

            // Spawn a new asynchronous task to handle the connection.
            // This allows the server to handle multiple connections concurrently.
            tokio::spawn(async move {
                // Write a "Hello!" message to the client.
                if let Err(e) = &mut sock.write_all("Hello!".as_bytes()).await {
                    // Log the error and panic if there is an issue writing the response.
                    error!("{}", e);
                    panic!("Error writing response")
                }
                // The connection is closed automatically when `sock` goes out of scope.
            });
        }
    }

    /// Accepts a new incoming TCP connection and returns the corresponding
    /// tokio TcpStream.
    async fn accept_conn(&mut self) -> Result<TcpStream> {
        loop {
            // Wait for an incoming connection.
            // The `accept()` method returns a tuple of (TcpStream, SocketAddr),
            // but we only need the TcpStream.
            match self.listener.accept().await {
                // Return the TcpStream if a connection is successfully accepted.
                Ok((sock, _)) => return Ok(sock),
                // Return an error if there is an issue accepting a connection.
                Err(e) => return Err(Error::from(e)),
            }
        }
    }
}
```

## The Main Function

Now, let's set up our [`main.rs`](http://main.rs) to use the above server. This main function does the following:

1. Initializes the logger.
2. Sets up the address and port for the server (127.0.0.1:6379).
3. Binds a `TcpListener` to this address.
4. Creates a new `Server` instance with this listener.
5. Runs the server.


```rust
// src/main.rs

// Include the server module defined in server.rs
mod server;

use crate::server::Server;
use anyhow::Result;
use log::info;
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> Result<()> {
    // Initialize the logger.
    // This sets up logging based on the RUST_LOG environment variable
    env_logger::init();

    // Define the address and port for the TCP server to listen on
    // Here we're using localhost (127.0.0.1) and port 6379 (commonly used for Redis)
    let addr = format!("127.0.0.1:{}", 6379);

    // Attempt to bind the TCP listener to the specified address and port
    let listener = match TcpListener::bind(&addr).await {
        // If successful, return the TcpListener
        Ok(tcp_listener) => {
            info!("TCP listener started on port 6379");
            tcp_listener
        },
        // If there is an error, panic and print the error message
        // This could happen if the port is already in use, for example
        Err(e) => panic!("Could not bind the TCP listener to {}. Err: {}", &addr, e),
    };

    // Create a new instance of the Server with the bound TcpListener
    let mut server = Server::new(listener);

    // Run the server to start accepting and handling connections
    // This will run indefinitely until the program is terminated
    server.run().await?;

    // This Ok(()) is technically unreachable as server.run() loops infinitely,
    // but it's needed to satisfy the Result return type of main()
    Ok(())
}
```

## Running the TCP server

Run the following command to run the application.

```bash
RUST_LOG=info cargo run
```

`RUST_LOG=info` will set the log level to `info`.

Now you can send TCP messages to the server using command-line utilities like `nc`.

```bash
nc -v localhost 6379
```

The above command will be responded with a `Hello!` from the TCP server.

## Source Code

The source code for this specific part is available at [https://github.com/dheerajgopi/nimblecache/tree/blog-1](https://github.com/dheerajgopi/nimblecache/tree/blog-1). Feel free to check the [main](https://github.com/dheerajgopi/nimblecache/tree/main) branch of the [Nimblecache](https://github.com/dheerajgopi/nimblecache) repository to see the latest code.

## Conclusion and what's next

This simple TCP server demonstrates the basics of network programming with Rust and Tokio. The next step is to expand this TCP server to handle the Redis Serialization Protocol (or RESP), enabling it to process and respond to Redis commands effectively. This will lay the groundwork for implementing more advanced features and functionalities of our Redis clone (Nimblecache). We'll cover this in the following part of the series.

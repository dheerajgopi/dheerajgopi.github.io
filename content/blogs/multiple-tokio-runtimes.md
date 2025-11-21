---
title: "How to Run Multiple Tokio Runtimes in a Rust Application"
date: 2024-09-11T00:00:00+05:30
draft: false
tags: ["rust", "tokio", "tcp"]
image: /images/blogs/banner-multiple-tokio-runtimes.webp
---

Getting started with Tokio is straightforward. By simply adding the `#[tokio::main]` macro to your entry point and using `tokio::spawn` for task management, you can quickly build an asynchronous application that handles typical use cases effectively.

However, as your application grows in complexity and you need finer control over thread allocation and performance, you'll want to move beyond the default setup and configure the Tokio runtime manually.

In this article, we'll explore a technique where multiple Tokio runtimes are used within a single application. Each runtime is configured with its own parameters and dedicated to specific types of tasks. For example, you might use one runtime to manage lightweight tasks, while another handles more resource-intensive operations.

To illustrate this approach, we'll walk through a practical example: a TCP echo server. In this setup, one Tokio runtime will be responsible for accepting incoming connections, while a separate runtime processes the TCP messages, demonstrating how this technique can help you balance workloads and improve overall application performance.

Here’s the code for a very basic echo server built using the `#[tokio::main]` macro. We’ll use this as a starting point.

```rust
// rustc version 1.77.2
// tokio version 1.38.0, features: full

use std::error::Error;

use tokio::{
    io::{AsyncReadExt, AsyncWriteExt},
    net::TcpListener,
};

/// This macro sets up a default Tokio runtime, and will execute the main method
/// within it.
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    let listener = TcpListener::bind("127.0.0.1:8000").await?;

    // loop for accepting incoming connections
    loop {
        let (mut sock, _) = listener.accept().await?;

        // Spawn a async task for each connection.
        // The task handles both read and write operations on the connection.
        tokio::spawn(async move {
            let mut buf = vec![0; 512];

            loop {
                let bytes_read = sock.read(&mut buf).await.expect("failed to read request");

                if bytes_read == 0 {
                    return;
                }

                buf.push(b'\n');

                sock.write_all(&buf[0..bytes_read + 1])
                    .await
                    .expect("failed to write response");

                sock.flush().await.expect("failed to flush response")
            }
        });
    }
}
```

## Echo Server With Multiple Tokio Runtimes

Instead of setting up a Tokio runtime using the macro, we will use `tokio::runtime::Builder` to do the same, but with much more control over its configuration.

This is an overview of what we’ll be doing:

* Create 2 Tokio runtimes: `acceptor_runtime` and `echo_runtime`.
* Create a `tokio::sync::mpsc` channel.
* `acceptor_runtime` tasks accept new TCP connections and send the `TcpStream` to the mpsc channel.
* `echo_runtime` tasks receive the `TcpStream` from the receiver half of the mpsc channel, and write the echo response.


Here’s the code for the above logic:

```rust
// rustc version 1.77.2
// tokio version 1.38.0, features: rt-multi-thread, net, io-util

use std::error::Error;

use tokio::{
    io::{AsyncReadExt, AsyncWriteExt},
    net::{TcpListener, TcpStream},
    sync::mpsc,
};

fn main() -> Result<(), Box<dyn Error>> {
    // Create a tokio runtime whose job is to simply accept new incoming TCP connections.
    let acceptor_runtime = tokio::runtime::Builder::new_multi_thread()
        .worker_threads(2)
        .thread_name("acceptor-pool")
        .enable_all()
        .build()?;

    // Create another tokio runtime whose job is only to write the response bytes to the outgoing TCP message.
    let echo_runtime = tokio::runtime::Builder::new_multi_thread()
        .worker_threads(8)
        .thread_name("echo-handler-pool")
        .enable_all()
        .build()?;

    // this channel is used to pass the TcpStream from acceptor_runtime task to
    // to echo_runtime task where the request handling is done.
    let (tx, mut rx) = mpsc::channel::<TcpStream>(4000);

    // The receiver part of the channel is moved inside a echo_runtime task.
    // This task simply writes the echo response to the TcpStreams coming through the
    // channel receiver.
    echo_runtime.spawn(async move {
        while let Some(mut sock) = rx.recv().await {
            tokio::spawn(async move {
                loop {
                    let mut buf = vec![0; 512];

                    loop {
                        let bytes_read = sock.read(&mut buf).await.expect("failed to read request");

                        if bytes_read == 0 {
                            return;
                        }

                        buf.push(b'\n');

                        sock.write_all(&buf[0..bytes_read + 1])
                            .await
                            .expect("failed to write response");
                    }
                }
            });
        }
    });

    // acceptor_runtime task is run in a blocking manner, so that our server starts
    // accepting new TCP connections. This task just accepts the incoming TcpStreams
    // and are sent to the sender half of the channel.
    acceptor_runtime.block_on(async move {
        let listener = match TcpListener::bind("127.0.0.1:8000").await {
            Ok(l) => l,
            Err(e) => panic!("error binding TCP listener: {}", e),
        };

        loop {
            let sock = match accept_conn(&listener).await {
                Ok(stream) => stream,
                Err(e) => panic!("error reading TCP stream: {}", e),
            };
            let _ = tx.send(sock).await;
        }
    });

    Ok(())
}

async fn accept_conn(listener: &TcpListener) -> Result<TcpStream, Box<dyn Error>> {
    loop {
        match listener.accept().await {
            Ok((sock, _)) => return Ok(sock),
            Err(e) => panic!("error accepting connection: {}", e),
        }
    }
}
```

## Advantages Over the Default Setup

Even though our code is slightly more complex, it has some advantages over the default macro setup.

* We can control the number of worker threads using the `worker_threads` option in `tokio::runtime::Builder` to better optimize the Tokio runtime based on the type of tasks it performs.
* Using a channel gives us better control over concurrency. We can easily build backpressure mechanisms with channels. Our default setup does not provide any control over the number of worker tasks spawned.
* The modified echo server has a better separation of concerns. It clearly separates the connection acceptance logic from the connection handling logic.
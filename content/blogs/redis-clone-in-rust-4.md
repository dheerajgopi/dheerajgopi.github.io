---
title: "Redis Clone in Rust - Part 4"
subtitle: "Handle Redis PING command"
date: 2024-08-29T00:00:00+05:30
draft: false
tags: ["rust", "redis", "tokio", "database"]
image: /images/blogs/banner-rust-redis-clone-4.png
---

In our journey to build a Redis clone using Rust and Tokio, we've laid the groundwork with a TCP server, a RESP parser and Redis command parsing capability. Let's now take it forward by implementing our first real Redis command: [PING](https://redis.io/docs/latest/commands/ping/).

The PING command is a simple operation in Redis, mainly used to test the connection between the client and the server. It responds with `PONG` if no argument is provided; otherwise, it echoes the argument back to the client.

## Implement PING command handler

Let's create a struct called `Ping` which handles the Redis PING command and returns the output RESP value. We will place the `Ping` struct inside a `command` Rust module.

Here's the general idea for whats being done in the below code:

* `Ping::with_args` accepts a vector of RespType values as arguments (remember that a Redis command is simply a list of RESP BulkString values), validates for the correct RESP data type, and creates a new instance of the `Ping` struct, which holds the optional message value.
* `apply` returns the response for the PING command. If an optional message is provided, it returns the same as a BulkString; otherwise, it returns the string `PONG` as a SimpleString.
* `Command` enum represents all the supported Redis commands. The `Ping` struct will be accessed as a variant of the `Command` enum.
* `Command::from_resp_command_frame` tries to parse the exact Redis command from a `Vec<RespType>` value. If parsing fails, it will return `CommandError`.

```rust
// src/command/ping.rs

use crate::resp::types::RespType;

use super::CommandError;

/// Represents the PING command in Nimblecache.
#[derive(Debug, Clone)]
pub struct Ping {
    /// Custom message
    msg: Option<String>,
}

impl Ping {
    /// Creates a new `Ping` instance from the given arguments.
    ///
    /// # Arguments
    ///
    /// * `args` - A vector of `RespType` representing the arguments to the PING command.
    ///
    /// # Returns
    ///
    /// * `Ok(Ping)` if parsing succeeds.
    /// * `Err(CommandError)` if parsing fails.
    pub fn with_args(args: Vec<RespType>) -> Result<Ping, CommandError> {
        if args.len() == 0 {
            return Ok(Ping { msg: None });
        }

        let msg = match &args[0] {
            RespType::BulkString(s) => s.clone(),
            _ => return Err(CommandError::Other(String::from("Invalid message"))),
        };

        Ok(Ping { msg: Some(msg) })
    }

    /// Executes the PING command.
    ///
    /// # Returns
    ///
    /// A `RespType` representing the response:
    /// - If no message was provided, it returns "PONG" as a `SimpleString`.
    /// - If a message was provided, it returns that message as a `BulkString`.
    pub fn apply(&self) -> RespType {
        if let Some(msg) = &self.msg {
            RespType::BulkString(msg.to_string())
        } else {
            RespType::SimpleString(String::from("PONG"))
        }
    }
}
```

```rust
// src/command/mod.rs

use core::fmt;

use ping::Ping;

use crate::resp::types::RespType;

mod ping;

/// Represents the supported Nimblecache commands.
#[derive(Debug, Clone)]
pub enum Command {
    /// The PING command.
    Ping(Ping),
}

impl Command {
    /// Attempts to parse a Nimblecache command from a RESP command frame.
    ///
    /// # Arguments
    ///
    /// * `frame` - A vector of `RespType` representing the command and its arguments.
    /// The first item is always the command name, and the rest are its arguments.
    ///
    /// # Returns
    ///
    /// * `Ok(Command)` if parsing succeeds.
    /// * `Err(CommandError)` if parsing fails.
    pub fn from_resp_command_frame(frame: Vec<RespType>) -> Result<Command, CommandError> {
        let (cmd_name, args) = frame.split_at(1);
        let cmd_name = match &cmd_name[0] {
            RespType::BulkString(s) => s.clone(),
            _ => return Err(CommandError::InvalidFormat),
        };

        let cmd = match cmd_name.to_lowercase().as_str() {
            "ping" => Command::Ping(Ping::with_args(Vec::from(args))?),
            _ => {
                return Err(CommandError::UnknownCommand(ErrUnknownCommand {
                    cmd: cmd_name,
                }));
            }
        };

        Ok(cmd)
    }

    /// Executes the Nimblecache command.
    ///
    /// # Returns
    ///
    /// The result of the command execution as a `RespType`.
    pub fn execute(&self) -> RespType {
        match self {
            Command::Ping(ping) => ping.apply(),
        }
    }
}

/// Represents all possible errors that can occur during command parsing and execution.
#[derive(Debug)]
pub enum CommandError {
    /// Indicates that the command format is invalid.
    InvalidFormat,
    /// Indicates that the command is unknown.
    UnknownCommand(ErrUnknownCommand),
    /// Represents any other error with a descriptive message.
    Other(String),
}

/// Represents an error for an unknown command.
#[derive(Debug)]
pub struct ErrUnknownCommand {
    /// The name of the unknown command.
    pub cmd: String,
}

impl std::error::Error for CommandError {}

impl fmt::Display for CommandError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            CommandError::InvalidFormat => "Invalid command format".fmt(f),
            CommandError::UnknownCommand(e) => write!(f, "Unknown command: {}", e.cmd),
            CommandError::Other(msg) => msg.as_str().fmt(f),
        }
    }
}
```

Now that we have the Ping command handler ready, let's update `src/handler.rs` to properly handle incoming Redis commands instead of just echoing them back to the client.

The below code changes do the following:

* Parse the exact `Command` type from the incoming Redis command.
* Execute the command and write the output RESP value into the outgoing TCP stream.
* If command execution fails, a RESP value of SimpleError type will be written into the outgoing TCP stream.

```diff
diff --git a/src/handler.rs b/src/handler.rs
index 8f2a002..2f6ae69 100644
--- a/src/handler.rs
+++ b/src/handler.rs
@@ -6,7 +6,10 @@ use log::error;
 use tokio::net::TcpStream;
 use tokio_util::codec::Framed;

-use crate::resp::{frame::RespCommandFrame, types::RespType};
+use crate::{
+    command::Command,
+    resp::{frame::RespCommandFrame, types::RespType},
+};

 /// Handles RESP command frames over a single TCP connection.
 pub struct FrameHandler {
@@ -22,7 +25,7 @@ impl FrameHandler {
     /// Handles incoming RESP command frames.
     ///
     /// This method continuously reads command frames from the connection,
-    /// and echo it back to the client. It continues until
+    /// processes them, and sends back the responses. It continues until
     /// an error occurs or the connection is closed.
     ///
     /// # Returns
@@ -37,8 +40,18 @@ impl FrameHandler {
         while let Some(resp_cmd) = self.conn.next().await {
             match resp_cmd {
                 Ok(cmd_frame) => {
+                    // Read the command from the frame.
+                    let resp_cmd = Command::from_resp_command_frame(cmd_frame);
+
+                    // Execute the command and get the RESP response.
+                    // If command fails, return RESP SimpleError as response.
+                    let response = match resp_cmd {
+                        Ok(cmd) => cmd.execute(),
+                        Err(e) => RespType::SimpleError(format!("{}", e)),
+                    };
+
                     // Write the RESP response into the TCP stream.
-                    if let Err(e) = self.conn.send(RespType::Array(cmd_frame)).await {
+                    if let Err(e) = self.conn.send(response).await {
                         error!("Error sending response: {}", e);
                         break;
                     }
```

## Setting server port using CLI parameter

This time the updated code will be tested using Redis CLI tool. But if you have Redis CLI tool in your machine, there's a good chance that Redis itself is running on port 6379. So, instead of instead of manually changing the port in code, let's add a CLI parameter to easily specify the port on which our Redis clone should run.

CLI parameters can be easily added to our application using the [clap](https://crates.io/crates/clap) crate.

The code changes for adding CLI parameter support are given below:

```diff
diff --git a/Cargo.toml b/Cargo.toml
index b50bb57..e0ac3a2 100644
--- a/Cargo.toml
+++ b/Cargo.toml
@@ -8,6 +8,7 @@ edition = "2021"
 [dependencies]
 anyhow = "1.0.86"
 bytes = "1.6.0"
+clap = { version = "4.5.8", features = ["derive"] }
 tokio = { version = "1.38.0", features = [
     "rt-multi-thread",
     "macros",
```

```diff
diff --git a/src/main.rs b/src/main.rs
index de0058f..59926d6 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -7,24 +7,44 @@ mod server;

 use crate::server::Server;
 use anyhow::Result;
+use clap::Parser;
 use log::info;
 use tokio::net::TcpListener;

+const DEFAULT_PORT: u16 = 6379;
+
+#[derive(Debug, Parser)]
+#[command(
+    name = "nimblecache-server",
+    version,
+    author,
+    about = "A RESP based in-memory cache"
+)]
+struct Cli {
+    /// Port to be bound to Nimblecache server
+    #[arg(long)]
+    port: Option<u16>,
+}
+
 #[tokio::main]
 async fn main() -> Result<()> {
     // Initialize the logger.
     // This sets up logging based on the RUST_LOG environment variable
     env_logger::init();

+    // Get port from --port CLI parameter. Defaults to 6379
+    let cli = Cli::parse();
+    let port = cli.port.unwrap_or(DEFAULT_PORT);
+
     // Define the address and port for the TCP server to listen on
     // Here we're using localhost (127.0.0.1) and port 6379 (commonly used for Redis)
-    let addr = format!("127.0.0.1:{}", 6379);
+    let addr = format!("127.0.0.1:{}", port);

     // Attempt to bind the TCP listener to the specified address and port
     let listener = match TcpListener::bind(&addr).await {
         // If successful, return the TcpListener
         Ok(tcp_listener) => {
-            info!("TCP listener started on port 6379");
+            info!("TCP listener started on port {}", port);
             tcp_listener
         }
         // If there is an error, panic and print the error message
```

Now you can use the below command to run the Redis clone on port 6380. If the `--port` parameter is not specified, it will default to 6379.

```bash
RUST_LOG=info cargo run -- --port 6380
```

## Testing PING using Redis CLI

Testing will be pretty simple now, since we are going to use the Redis CLI tool. Start the server in the port of your choice, and start the Redis CLI tool by specifying the same port as the server.

Next, issue a `PING` command in the CLI tool and check that our server responds with a `PONG`. Also, try sending a custom message with the PING command to see if the server responds with your custom message instead of `PONG`.

## Source Code

The source code for this specific part is available at [https://github.com/dheerajgopi/nimblecache/tree/blog-4](https://github.com/dheerajgopi/nimblecache/tree/blog-4).

If you are interested in seeing the git-diff between this part and the previous part of this series, have a look at this commit: [https://github.com/dheerajgopi/nimblecache/commit/92339123d878c30ce397b852cb8b8d1cc95cfc70](https://github.com/dheerajgopi/nimblecache/commit/92339123d878c30ce397b852cb8b8d1cc95cfc70)

Feel free to check the [main](https://github.com/dheerajgopi/nimblecache/tree/main) branch of the [Nimblecache](https://github.com/dheerajgopi/nimblecache/tree/main) repository to see the latest code.

## Conclusion and what's next

In this part, we successfully implemented the PING command handler for our Redis clone, allowing us to test the connection and echo custom messages. We also added CLI parameter support to easily specify the server port, making testing more flexible.

The next part will focus on implementing the storage mechanism and the basic SET and GET commands. This will enable our Redis clone to store and retrieve data, bringing us closer to a fully functional Redis-like server.

Stay tuned!
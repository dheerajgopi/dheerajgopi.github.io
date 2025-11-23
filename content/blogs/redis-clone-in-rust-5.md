---
title: "Redis Clone in Rust - Part 5"
subtitle: "Building key-value storage for a Redis clone"
date: 2024-09-02T00:00:00+05:30
draft: false
tags: ["rust", "redis", "tokio", "database"]
image: /images/blogs/banner-rust-redis-clone-5.png
---

In the previous part of this series, we explored how to handle a basic PING command in our Redis clone. In this part, we will implement the Redis SET and GET commands. Additionally, we will build a simple HashMap-based storage system to support these operations.

## Hash-map based storage

The storage system we're building for our Redis clone is conceptually very simple. it's just a global HashMap that employs thread-safe mechanisms to manage concurrent access. However, what might be straightforward to implement in other programming languages becomes slightly complex in Rust due to its strict ownership and concurrency model. In Rust, we have to handle shared state across threads using tools like `Arc` and `Mutex` or `RwLock`. These ensure that our data remains safe and consistent across multiple threads.

Check the code below for the HashMap storage implementation:

The rustdocs will help you understand each part of the code.

```rust
// src/storage/db.rs

use std::{
    collections::HashMap,
    sync::{Arc, RwLock},
};

use super::DBError;

/// The Storage struct is designed to act as a wrapper around the core database,
/// allowing it to be shared across multiple connections. The database is encapsulated within an Arc,
/// to enable concurrent access.
#[derive(Debug, Clone)]
pub struct Storage {
    db: Arc<DB>,
}

/// The DB struct is the component that houses the actual data,
/// which is stored in a RwLock wrapped around a HashMap. This ensures thread-safe read and write operations.
#[derive(Debug)]
pub struct DB {
    data: RwLock<HashMap<String, Entry>>,
}

/// The Entry struct represents the value associated with a particular key in the database.
/// This struct encapsulates the Value enum, which allows for different types of data to be stored.
#[derive(Debug, Clone)]
pub struct Entry {
    value: Value,
}

/// The `Value` enum allows for storing various types of data associated with a key.
/// Currently, it supports only String data type. But it can be expanded in the future
/// to support more data types as needed (like List, Hash etc).
#[derive(Debug, Clone)]
pub enum Value {
    String(String),
}

impl Storage {
    /// Create a new instance of `Storage` which contains the DB.
    pub fn new(db: DB) -> Storage {
        Storage { db: Arc::new(db) }
    }

    /// Returns a clone of the shared database (`Arc<DB>`).
    ///
    /// This method provides access to the underlying database, which is shared across all
    /// connections. The database is wrapped in an `Arc` to ensure concurrent access by multiple threads.
    pub fn db(&self) -> Arc<DB> {
        self.db.clone()
    }
}

impl DB {
    /// Create a new instance of DB.
    pub fn new() -> DB {
        DB {
            data: RwLock::new(HashMap::new()),
        }
    }

    /// Get the string value stored against a key.
    ///
    /// # Arguments
    ///
    /// * `k` - The key on which lookup is performed.
    ///
    /// # Returns
    ///
    /// * `Ok(Option<String>)` - `Some(String)` if key is found in DB, else `None`
    /// * `Err(DBError)` - if key already exists and has non-string data.
    pub fn get(&self, k: &str) -> Result<Option<String>, DBError> {
        let data = match self.data.read() {
            Ok(data) => data,
            Err(e) => return Err(DBError::Other(format!("{}", e))),
        };

        let entry = match data.get(k) {
            Some(entry) => entry,
            None => return Ok(None),
        };

        if let Value::String(s) = &entry.value {
            return Ok(Some(s.to_string()));
        }

        Err(DBError::WrongType)
    }

    /// Set a string value against a key.
    ///
    /// # Arguments
    ///
    /// * `k` - The key on which value is to be set.
    ///
    /// * `v` - The value to be set against the key.
    ///
    /// # Returns
    ///
    /// * `Ok(())` - If value is successfully added against the key.
    /// * `Err(DBError)` - if key already exists and has non-string data.
    pub fn set(&self, k: String, v: Value) -> Result<(), DBError> {
        let mut data = match self.data.write() {
            Ok(data) => data,
            Err(e) => return Err(DBError::Other(format!("{}", e))),
        };

        let entry = match data.get(k.as_str()) {
            Some(entry) => Some(entry),
            None => None,
        };

        if entry.is_some() {
            match entry.unwrap().value {
                Value::String(_) => {}
                _ => return Err(DBError::WrongType),
            }
        }

        data.insert(k.to_string(), Entry::new(v));

        return Ok(());
    }
}

impl Entry {
    pub fn new(value: Value) -> Entry {
        Entry { value }
    }
}
```

```rust
// src/storage/mod.rs

pub mod db;

/// Represents errors that can occur during DB operations.
#[derive(Debug)]
pub enum DBError {
    /// Represents an error where wrong data type is encountered against a key.
    /// For e.g. If you try to perform list related operation (such as lpush, rpush) on a key
    /// which stores a string value.
    WrongType,
    /// Represents any other error with a descriptive message.
    Other(String),
}

impl std::fmt::Display for DBError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            DBError::WrongType => {
                "WRONGTYPE Operation against a key holding the wrong kind of value".fmt(f)
            }
            DBError::Other(msg) => msg.as_str().fmt(f),
        }
    }
}
```

## Implementing SET and GET commands

Now that we've implemented a basic database, let's perform simple read and write operations using the SET and GET commands. The structure for implementing these commands is similar to how the PING command was implemented. However, the key difference is that the `apply` method for the SET and GET commands requires a reference to the storage layer, allowing them to interact directly with the database to store and retrieve data.

### SET command

Our implementation of SET command is very basic. Advanced options like TTL are ignored for now.

Here's the Redis documentation for the SET command: [https://redis.io/docs/latest/commands/set/](https://redis.io/docs/latest/commands/set/)

```rust
// src/command/set.rs

use crate::{
    resp::types::RespType,
    storage::db::{Value, DB},
};

use super::CommandError;

/// Represents the SET command in Nimblecache.
///
/// The `Set` struct encapsulates the key-value pair for the SET command, which is used
/// to store a string value against a key in the database.
#[derive(Debug, Clone)]
pub struct Set {
    key: String,
    value: String,
}

impl Set {
    /// Creates a new `Set` instance from the given arguments.
    ///
    /// This function parses the arguments provided in the form of a `RespType` vector.
    /// It validates and extracts the key and value for the SET command.
    ///
    /// # Arguments
    ///
    /// * `args` - A vector of `RespType` representing the arguments to the SET command.
    ///
    /// # Returns
    ///
    /// * `Ok(Set)` - If parsing succeeds and the key-value pair is valid.
    /// * `Err(CommandError)` - if parsing fails due to validation errors.
    pub fn with_args(args: Vec<RespType>) -> Result<Set, CommandError> {
        if args.len() < 2 {
            return Err(CommandError::Other(String::from(
                "Wrong number of arguments specified for 'SET' command",
            )));
        }

        // parse key
        let key = &args[0];
        let key = match key {
            RespType::BulkString(k) => k,
            _ => {
                return Err(CommandError::Other(String::from(
                    "Invalid argument. Key must be a bulk string",
                )));
            }
        };

        // parse value
        let value = &args[1];
        let value = match value {
            RespType::BulkString(v) => v.to_string(),
            _ => {
                return Err(CommandError::Other(String::from(
                    "Invalid argument. Value must be a bulk string",
                )));
            }
        };

        Ok(Set {
            key: key.to_string(),
            value,
        })
    }

    /// Executes the SET command.
    ///
    /// This method writes the string value to the database under the specified key.
    /// If the operation is successful, it returns an "OK" response as a `BulkString`.
    /// If the operation fails, it returns an error response.
    ///
    /// # Arguments
    ///
    /// * `db` - A reference to the `DB` instance where the key-value pair should be stored.
    ///
    /// # Returns
    ///
    /// * `BulkString("OK")` - If the value is successfully written.
    /// * `SimpleError` - If the operation fails due to some error.
    pub fn apply(&self, db: &DB) -> RespType {
        match db.set(self.key.clone(), Value::String(self.value.clone())) {
            Ok(_) => RespType::BulkString("OK".to_string()),
            Err(e) => RespType::SimpleError(format!("{}", e)),
        }
    }
}
```

### GET command

Here's the Redis documentation for GET command: [https://redis.io/docs/latest/commands/get/](https://redis.io/docs/latest/commands/get/)

If you read the above documentation, you'll find that Redis responds with a `nil` if the specified key does not exist. But as of now, our Redis clone does not have a corresponding `RespType` value for `nil`. Let's implement that too as part of building the GET command handler.

`nil` values are represented in RESP using `NullBulkString`. It's simply a BulkString with length of negative one (-1).

```diff
diff --git a/src/resp/types.rs b/src/resp/types.rs
index ebedb2c..9dd2930 100644
--- a/src/resp/types.rs
+++ b/src/resp/types.rs
@@ -5,6 +5,8 @@ pub enum RespType {
     SimpleString(String),
     /// Refer <https://redis.io/docs/latest/develop/reference/protocol-spec/#bulk-strings>
     BulkString(String),
+    /// Null representation in RESP2. It's simply a BulkString with length of negative one (-1).
+    NullBulkString,
     /// Refer <https://redis.io/docs/latest/develop/reference/protocol-spec/#simple-errors>
     SimpleError(String),
     /// Refer <https://redis.io/docs/latest/develop/reference/protocol-spec/#arrays>
@@ -127,6 +129,7 @@ impl RespType {
                 let bulkstr_bytes = format!("${}\r\n{}\r\n", bs.chars().count(), bs).into_bytes();
                 Bytes::from_iter(bulkstr_bytes)
             }
+            RespType::NullBulkString => Bytes::from("$-1\r\n"),
             RespType::Array(arr) => {
                 let mut arr_bytes = format!("*{}\r\n", arr.len()).into_bytes();
                 arr.iter()
```

Now lets implement the GET command handler.

```rust
// src/command/get.rs

use crate::{resp::types::RespType, storage::db::DB};

use super::CommandError;

/// Represents the GET command in Nimblecache.
///
/// The `Get` struct is used to retrieve the value associated with a specified key
/// from the database.
#[derive(Debug, Clone)]
pub struct Get {
    /// Key to be searched in the database
    key: String,
}

impl Get {
    /// Creates a new `Get` instance from the given arguments.
    ///
    /// This function parses the arguments provided in the form of a `RespType` vector.
    /// It validates and extracts the key for the GET command.
    ///
    /// # Arguments
    ///
    /// * `args` - A vector of `RespType` representing the arguments to the GET command.
    ///
    /// # Returns
    ///
    /// * `Ok(Get)` - If parsing succeeds and the key is valid.
    /// * `Err(CommandError)` - if parsing fails due to validation errors.
    pub fn with_args(args: Vec<RespType>) -> Result<Get, CommandError> {
        if args.len() < 1 {
            return Err(CommandError::Other(String::from(
                "Wrong number of arguments specified for 'GET' command",
            )));
        }

        // parse key
        let key = &args[0];
        let key = match key {
            RespType::BulkString(k) => k.to_string(),
            _ => {
                return Err(CommandError::Other(String::from(
                    "Invalid argument. Key must be a bulk string",
                )));
            }
        };

        Ok(Get { key })
    }

    /// Executes the GET command.
    ///
    /// # Arguments
    ///
    /// * `db` - The database where the key and values are stored.
    ///
    /// # Returns
    ///
    /// - If key is present in DB - Value of the key as a `BulkString`
    /// - If key is not found in DB - A `NullBulkString`
    /// - If an error is encountered - A `SimpleError` with an error message
    pub fn apply(&self, db: &DB) -> RespType {
        match db.get(self.key.as_str()) {
            Ok(val) => match val {
                Some(s) => RespType::BulkString(s),
                None => RespType::NullBulkString,
            },
            Err(e) => RespType::SimpleError(format!("{}", e)),
        }
    }
}
```

Now, let's add the SET and GET command handlers to the `Command` enum.

```diff
diff --git a/src/command/mod.rs b/src/command/mod.rs
index eb914ac..ba4b8b4 100644
--- a/src/command/mod.rs
+++ b/src/command/mod.rs
@@ -2,17 +2,25 @@

 use core::fmt;

+use get::Get;
 use ping::Ping;
+use set::Set;

-use crate::resp::types::RespType;
+use crate::{resp::types::RespType, storage::db::DB};

+mod get;
 mod ping;
+mod set;

 /// Represents the supported Nimblecache commands.
 #[derive(Debug, Clone)]
 pub enum Command {
     /// The PING command.
     Ping(Ping),
+    /// The SET command.
+    Set(Set),
+    /// The GET command.
+    Get(Get),
 }

 impl Command {
@@ -36,6 +44,20 @@ impl Command {

         let cmd = match cmd_name.to_lowercase().as_str() {
             "ping" => Command::Ping(Ping::with_args(Vec::from(args))?),
+            "set" => {
+                let cmd = Set::with_args(Vec::from(args));
+                match cmd {
+                    Ok(cmd) => Command::Set(cmd),
+                    Err(e) => return Err(e),
+                }
+            }
+            "get" => {
+                let cmd = Get::with_args(Vec::from(args));
+                match cmd {
+                    Ok(cmd) => Command::Get(cmd),
+                    Err(e) => return Err(e),
+                }
+            }
             _ => {
                 return Err(CommandError::UnknownCommand(ErrUnknownCommand {
                     cmd: cmd_name,
@@ -48,12 +70,18 @@ impl Command {

     /// Executes the Nimblecache command.
     ///
+    /// # Arguments
+    ///
+    /// * `db` - Reference to the database where the key-value pairs are stored.
+    ///
     /// # Returns
     ///
     /// The result of the command execution as a `RespType`.
-    pub fn execute(&self) -> RespType {
+    pub fn execute(&self, db: &DB) -> RespType {
         match self {
             Command::Ping(ping) => ping.apply(),
+            Command::Set(set) => set.apply(db),
+            Command::Get(get) => get.apply(db),
         }
     }
 }
```

### Other code changes

Below are some additional code changes required to pass the DB reference to the command handlers:

```diff
diff --git a/src/handler.rs b/src/handler.rs
index 2f6ae69..5ee044c 100644
--- a/src/handler.rs
+++ b/src/handler.rs
@@ -9,6 +9,7 @@ use tokio_util::codec::Framed;
 use crate::{
     command::Command,
     resp::{frame::RespCommandFrame, types::RespType},
+    storage::db::DB,
 };

 /// Handles RESP command frames over a single TCP connection.
@@ -28,6 +29,10 @@ impl FrameHandler {
     /// processes them, and sends back the responses. It continues until
     /// an error occurs or the connection is closed.
     ///
+    /// # Arguments
+    ///
+    /// * `db` - Reference to the database where the key-value pairs are stored.
+    ///
     /// # Returns
     ///
     /// A `Result` indicating whether the operation succeeded or failed.
@@ -36,7 +41,7 @@ impl FrameHandler {
     ///
     /// This method will return an error if there's an issue with reading
     /// from or writing to the connection.
-    pub async fn handle(mut self) -> Result<()> {
+    pub async fn handle(mut self, db: &DB) -> Result<()> {
         while let Some(resp_cmd) = self.conn.next().await {
             match resp_cmd {
                 Ok(cmd_frame) => {
@@ -46,7 +51,7 @@ impl FrameHandler {
                     // Execute the command and get the RESP response.
                     // If command fails, return RESP SimpleError as response.
                     let response = match resp_cmd {
-                        Ok(cmd) => cmd.execute(),
+                        Ok(cmd) => cmd.execute(db),
                         Err(e) => RespType::SimpleError(format!("{}", e)),
                     };
```

```diff
diff --git a/src/server.rs b/src/server.rs
index 9e45bee..5fe3af2 100644
--- a/src/server.rs
+++ b/src/server.rs
@@ -1,5 +1,7 @@
 // src/server.rs

+use std::sync::Arc;
+
 // anyhow provides the Error and Result types for convenient error handling
 use anyhow::{Error, Result};

@@ -9,24 +11,33 @@ use log::error;
 use tokio::net::{TcpListener, TcpStream};
 use tokio_util::codec::Framed;

-use crate::{handler::FrameHandler, resp::frame::RespCommandFrame};
+use crate::{handler::FrameHandler, resp::frame::RespCommandFrame, storage::db::Storage};

-/// The Server struct holds the tokio TcpListener which listens for
-/// incoming TCP connections.
+/// The Server struct holds:
+///
+/// * the tokio TcpListener which listens for incoming TCP connections.
+///
+/// * Shared storage
+///
 #[derive(Debug)]
 pub struct Server {
+    /// The TCP listener for accepting incoming connections.
     listener: TcpListener,
+    /// Contains the shared storage.
+    storage: Storage,
 }

 impl Server {
-    /// Creates a new Server instance with the given TcpListener.
-    pub fn new(listener: TcpListener) -> Server {
-        Server { listener }
+    /// Creates a new Server instance with the given TcpListener and shared storage.
+    pub fn new(listener: TcpListener, storage: Storage) -> Server {
+        Server { listener, storage }
     }

     /// Runs the server in an infinite loop, continuously accepting and handling
     /// incoming connections.
     pub async fn run(&mut self) -> Result<()> {
+        let db = self.storage.db().clone();
+
         loop {
             // accept a new TCP connection.
             // If successful the corresponding TcpStream is stored
@@ -44,11 +55,14 @@ impl Server {
             // and to write RespType values into outgoing TCP messages.
             let resp_command_frame = Framed::with_capacity(sock, RespCommandFrame::new(), 8 * 1024);

+            // Clone the Arc of DB for passing it to the tokio task.
+            let db = Arc::clone(&db);
+
             // Spawn a new asynchronous task to handle the connection.
             // This allows the server to handle multiple connections concurrently.
             tokio::spawn(async move {
                 let handler = FrameHandler::new(resp_command_frame);
-                if let Err(e) = handler.handle().await {
+                if let Err(e) = handler.handle(db.as_ref()).await {
                     error!("Failed to handle command: {}", e);
                 }
             });
```

```diff
diff --git a/src/main.rs b/src/main.rs
index 59926d6..ca4e16b 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -4,6 +4,7 @@ mod command;
 mod handler;
 mod resp;
 mod server;
+mod storage;

 use crate::server::Server;
 use anyhow::Result;
@@ -52,8 +53,11 @@ async fn main() -> Result<()> {
         Err(e) => panic!("Could not bind the TCP listener to {}. Err: {}", &addr, e),
     };

+    // initialize shared storage
+    let shared_storage = storage::db::Storage::new(storage::db::DB::new());
+
     // Create a new instance of the Server with the bound TcpListener
-    let mut server = Server::new(listener);
+    let mut server = Server::new(listener, shared_storage);

     // Run the server to start accepting and handling connections
     // This will run indefinitely until the program is terminated
```

## Testing SET and GET

Start Nimblecache using the below command and use Redis client to connect to it.

```bash
# --port parameter is optional
RUST_LOG=info cargo run -- --port 6380
```

Now, try issuing SET and GET commands to see if they work properly.

## Source code

The source code for this specific part is available at [https://github.com/dheerajgopi/nimblecache/tree/blog-5](https://github.com/dheerajgopi/nimblecache/tree/blog-5).

If you are interested in seeing the git-diff between this part and the previous part of this series, have a look at this commit: [https://github.com/dheerajgopi/nimblecache/commit/0d7543f4305a7c187319b1baa6a20a7b683ced7a](https://github.com/dheerajgopi/nimblecache/commit/0d7543f4305a7c187319b1baa6a20a7b683ced7a)

Feel free to check the [main](https://github.com/dheerajgopi/nimblecache/tree/main) branch of the [Nimblecache](https://github.com/dheerajgopi/nimblecache/tree/main) repository to see the latest code.

## Conclusion and what's next

In this part, we successfully implemented the SET and GET commands for our Redis clone, along with a simple HashMap-based storage system. This allows us to perform basic key-value operations, laying the foundation for more complex features.

In the next part of this series, we will implement the Redis LIST data type and its related operations, enabling us to store and manipulate lists within our Redis clone.

Stay tuned!
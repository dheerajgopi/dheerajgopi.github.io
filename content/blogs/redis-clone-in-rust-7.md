---
title: "Redis Clone in Rust - Part 7"
subtitle: "Implement Transaction Support in a Redis Clone"
date: 2024-09-17T00:00:00+05:30
draft: false
tags: ["rust", "redis", "tokio", "database"]
image: /images/blogs/banner-rust-redis-clone-7.png
---

Redis transactions provide a mechanism to execute a group of commands as a single atomic operation. When a transaction is initiated, subsequent commands are queued rather than executed immediately. Once all desired commands are added to the transaction, it can be executed or aborted. During execution, Redis guarantees that all commands in the transaction will be processed sequentially and atomically, without interruption from other clients. However, Redis transactions differ from traditional database transactions in important ways. They don't support rollbacks - if a command fails during execution, other commands in the transaction will still be processed.

## MULTI, EXEC and DISCARD

A transaction begins with the MULTI command, which tells Redis that subsequent commands should be queued rather than executed immediately. After MULTI, each command is validated but not executed, and Redis responds with "QUEUED". Once all desired commands are queued, the EXEC command is used to execute the entire transaction. The DISCARD command can be used after MULTI but before EXEC to cancel the transaction and clear the queued commands.

Let’s start with the code changes now. The transaction logic is enclosed in a `Transaction` struct, which has the below methods:

* `init` - Initializes a transaction.
* `add_command` - Queues a command into the transaction.
* `exec` - Executes all commands inside the transaction sequentially.
* `discard` - Discards all commands inside the transaction.

```rust
// src/command/transactions.rs

use crate::{resp::types::RespType, storage::db::DB};

use super::Command;

/// Represents a Redis transaction that can be executed atomically (MULTI and EXEC).
pub struct Transaction {
    /// The queue of commands to be executed.
    commands: Vec<Command>,
    /// Indicates whether a transaction is currently active.
    is_active: bool,
}

impl Transaction {
    /// Creates a new `Transaction` instance.
    pub fn new() -> Transaction {
        Transaction {
            commands: vec![],
            is_active: false,
        }
    }

    /// Initializes a new transaction (MULTI command).
    ///
    /// # Returns
    ///
    /// * `Ok(())` if the transaction was successfully initialized.
    /// * `Err(TransactionError::CannotNestMulti)` if a transaction is already active.
    pub fn init(&mut self) -> Result<(), TransactionError> {
        if self.is_active {
            return Err(TransactionError::CannotNestMulti);
        }
        self.is_active = true;

        Ok(())
    }

    /// Adds a command to the transaction.
    ///
    /// # Arguments
    ///
    /// * `cmd` - The command to be added to the transaction.
    pub fn add_command(&mut self, cmd: Command) {
        self.commands.push(cmd);
    }

    /// Checks if a transaction is currently active.
    pub fn is_active(&self) -> bool {
        self.is_active
    }

    /// Executes the commands in the transaction and returns the array of responses.
    ///
    /// This method will execute all the commands in the transaction and return the
    /// responses as a `RespType::Array`. After the execution, the transaction is
    /// automatically discarded.
    ///
    /// # Arguments
    ///
    /// * `db` - The database where the key and values are stored.
    ///
    /// # Returns
    ///
    /// A `RespType::Array` containing the responses for each command in the transaction.
    pub async fn exec(&mut self, db: &DB) -> RespType {
        let mut responses: Vec<RespType> = vec![];

        for cmd in self.commands.iter() {
            // execute the command
            let res = cmd.execute(db);

            responses.push(res);
        }

        // discard txn after executing all commands
        self.discard();

        RespType::Array(responses)
    }

    /// Discards the current transaction.
    ///
    /// This method clears the queue of commands and resets the `is_active` flag.
    pub fn discard(&mut self) {
        self.commands = vec![];
        self.is_active = false;
    }
}

/// Represents errors that can occur during transaction operations.
#[derive(Debug)]
pub enum TransactionError {
    /// Indicates that a MULTI command cannot be nested within another active transaction.
    CannotNestMulti,
}

impl std::error::Error for TransactionError {}

impl std::fmt::Display for TransactionError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            TransactionError::CannotNestMulti => "MULTI calls cannot be nested".fmt(f),
        }
    }
}
```

Now lets add the MULTI, EXEC and DISCARD commands to our `Command` enum.

```diff
diff --git a/src/command/mod.rs b/src/command/mod.rs
index 71196fc..2804775 100644
--- a/src/command/mod.rs
+++ b/src/command/mod.rs
@@ -17,6 +17,7 @@ mod lrange;
 mod ping;
 mod rpush;
 mod set;
+pub mod transactions;

 /// Represents the supported Nimblecache commands.
 #[derive(Debug, Clone)]
@@ -33,6 +34,12 @@ pub enum Command {
     RPush(RPush),
     /// The LRANGE command.
     LRange(LRange),
+    /// The MULTI command.
+    Multi,
+    /// The EXEC command.
+    Exec,
+    /// The DISCARD command.
+    Discard,
 }

 impl Command {
@@ -91,6 +98,9 @@ impl Command {
                     Err(e) => return Err(e),
                 }
             }
+            "multi" => Command::Multi,
+            "exec" => Command::Exec,
+            "discard" => Command::Discard,
             _ => {
                 return Err(CommandError::UnknownCommand(ErrUnknownCommand {
                     cmd: cmd_name,
@@ -118,6 +128,12 @@ impl Command {
             Command::LPush(lpush) => lpush.apply(db),
             Command::RPush(rpush) => rpush.apply(db),
             Command::LRange(lrange) => lrange.apply(db),
+            // MULTI calls are handled inside FrameHandler.handle since it involves command queueing.
+            Command::Multi => RespType::SimpleString(String::from("OK")),
+            // EXEC calls are handled inside FrameHandler.handle too, since it involves executing queued commands.
+            Command::Exec => RespType::NullBulkString,
+            // DISCARD calls are handled inside FrameHandler.handle too, since it involves discarding queued commands.
+            Command::Discard => RespType::SimpleString(String::from("OK")),
         }
     }
 }
```

You’ll notice that the commands simply return an `OK` or `nil`, and there’s no handler attached to each command. This is because once a transaction is started using the MULTI command, all following commands should be queued. Therefore, MULTI, EXEC, and DISCARD are handled as special cases in our `FrameHandler.handle` method.

```diff
diff --git a/src/handler.rs b/src/handler.rs
index 5ee044c..9bbe529 100644
--- a/src/handler.rs
+++ b/src/handler.rs
@@ -7,7 +7,7 @@ use tokio::net::TcpStream;
 use tokio_util::codec::Framed;

 use crate::{
-    command::Command,
+    command::{transactions::Transaction, Command},
     resp::{frame::RespCommandFrame, types::RespType},
     storage::db::DB,
 };
@@ -29,6 +29,20 @@ impl FrameHandler {
     /// processes them, and sends back the responses. It continues until
     /// an error occurs or the connection is closed.
     ///
+    /// The server's behavior depends on whether a `MULTI` command has been issued.
+    ///
+    /// ## No MULTI Command Issued
+    ///
+    /// If no `MULTI` command has been issued, each command is executed
+    /// immediately and its response is sent back.
+    ///
+    /// ## MULTI Command Issued
+    ///
+    /// If a `MULTI` command has been issued, the method will enter a transaction
+    /// mode. In this mode, all subsequent commands will be queued until an
+    /// `EXEC` command is received. When `EXEC` is called, all the queued
+    /// commands are executed, and the array of responses is sent back.
+    ///
     /// # Arguments
     ///
     /// * `db` - Reference to the database where the key-value pairs are stored.
@@ -42,17 +56,59 @@ impl FrameHandler {
     /// This method will return an error if there's an issue with reading
     /// from or writing to the connection.
     pub async fn handle(mut self, db: &DB) -> Result<()> {
+        // commands are queued here if MULTI command was issued
+        let mut multicommand = Transaction::new();
+
         while let Some(resp_cmd) = self.conn.next().await {
             match resp_cmd {
                 Ok(cmd_frame) => {
                     // Read the command from the frame.
                     let resp_cmd = Command::from_resp_command_frame(cmd_frame);

-                    // Execute the command and get the RESP response.
-                    // If command fails, return RESP SimpleError as response.
+                    // If command is parsed successfully, execute it and get the RESP response,
+                    // otherwise set a SimpleError RESP value as the response.
                     let response = match resp_cmd {
-                        Ok(cmd) => cmd.execute(db),
-                        Err(e) => RespType::SimpleError(format!("{}", e)),
+                        Ok(cmd) => match cmd {
+                            // Initialize pipeline if MULTI command is issued
+                            Command::Multi => {
+                                let init_multicommand = &mut multicommand.init();
+                                match init_multicommand {
+                                    Ok(_) => cmd.execute(db),
+                                    Err(e) => RespType::SimpleError(format!("{}", e)),
+                                }
+                            }
+                            // Execute all commands in pipeline if EXEC command is issued
+                            Command::Exec => {
+                                if multicommand.is_active() {
+                                    multicommand.exec(db).await
+                                } else {
+                                    RespType::SimpleError(String::from("EXEC without MULTI"))
+                                }
+                            }
+                            Command::Discard => {
+                                if multicommand.is_active() {
+                                    multicommand.discard();
+                                    cmd.execute(db)
+                                } else {
+                                    RespType::SimpleError(String::from("DISCARD without MULTI"))
+                                }
+                            }
+                            _ => {
+                                // Queue commands if pipeline is active, else execute the command
+                                if multicommand.is_active() {
+                                    multicommand.add_command(cmd);
+                                    RespType::SimpleString(String::from("QUEUED"))
+                                } else {
+                                    cmd.execute(db)
+                                }
+                            }
+                        },
+                        Err(e) => {
+                            if multicommand.is_active() {
+                                multicommand.discard();
+                            }
+                            RespType::SimpleError(format!("{}", e))
+                        }
                     };

                     // Write the RESP response into the TCP stream.
```

## **Testing**

Start Nimblecache using the below command and use Redis client to connect to it.

```diff
# --port parameter is optional
RUST_LOG=info cargo run -- --port 6380
```

Now run the below commands:

* `MULTI`
* `SET a 1` - At this point, if you connect to Nimblecache using redis-cli in a different terminal and execute `GET a`, you’ll get `nil` as the response.
* `EXEC` - All queued commands will be executed now. If you execute `GET a` now, you’ll get `1` as the response.

## **Source code**

The source code for this specific part is available at [https://github.com/dheerajgopi/nimblecache/tree/blog-7](https://github.com/dheerajgopi/nimblecache/tree/blog-7).

If you are interested in seeing the git-diff between this part and the previous part of this series, have a look at this commit: [https://github.com/dheerajgopi/nimblecache/commit/42a4f164ec1ba683e6fc7dd5d652e315e149e63a](https://github.com/dheerajgopi/nimblecache/commit/42a4f164ec1ba683e6fc7dd5d652e315e149e63a)

Feel free to check the [**main**](https://github.com/dheerajgopi/nimblecache/tree/main) branch of the [**Nimblecache**](https://github.com/dheerajgopi/nimblecache/tree/main) repository to see the latest code.
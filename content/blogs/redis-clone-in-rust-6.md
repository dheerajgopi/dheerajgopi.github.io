---
title: "Redis Clone in Rust - Part 6"
subtitle: "Implement List Data Type for a Redis Clone"
date: 2024-09-09T00:00:00+05:30
draft: false
tags: ["rust", "redis", "tokio", "database"]
image: /images/blogs/banner-rust-redis-clone-6.png
---

In our ongoing series on building a Redis clone using Rust and Tokio, we've already set up a key-value storage and implemented the essential GET and SET commands. Now, it's time to introduce a more complex data structure: the LIST.

Before proceeding further, take a look at the official Redis documentation: [https://redis.io/docs/latest/develop/data-types/lists/](https://redis.io/docs/latest/develop/data-types/lists/).

Please note that not all List commands will be implemented in this exercise. We'll cover only LPUSH, RPUSH, and LRANGE.

## Add Support for List Data Type

Building upon our existing key-value storage system, implementing the LIST data type in our Redis clone is surprisingly straightforward. We'll leverage Rust's standard library, specifically the `VecDeque` (double-ended queue) data structure, to efficiently handle operations at both ends of the list. `VecDeque` is an ideal choice for our implementation as it provides O(1) time complexity for insertions and deletions at both the front and back of the list. This aligns perfectly with Redis' LIST operations like LPUSH, RPUSH, LPOP, and RPOP, which primarily manipulate the head and tail of the list. By using `VecDeque`, we can ensure that our LIST commands will perform optimally, mirroring the efficiency users expect from Redis.

Below are the required code changes:

```diff
 diff --git a/src/storage/db.rs b/src/storage/db.rs
index 187965a..c567b48 100644
--- a/src/storage/db.rs
+++ b/src/storage/db.rs
@@ -1,7 +1,7 @@
 // src/storage/db.rs

 use std::{
-    collections::HashMap,
+    collections::{HashMap, VecDeque},
     sync::{Arc, RwLock},
 };

@@ -30,11 +30,12 @@ pub struct Entry {
 }

 /// The `Value` enum allows for storing various types of data associated with a key.
-/// Currently, it supports only String data type. But it can be expanded in the future
-/// to support more data types as needed (like List, Hash etc).
+/// Currently, it supports only String and List data type. But it can be expanded in the future
+/// to support more data types as needed (like Hash, SortedSet etc).
 #[derive(Debug, Clone)]
 pub enum Value {
     String(String),
+    List(VecDeque<String>),
 }

 impl Storage {
@@ -122,6 +123,184 @@ impl DB {

         return Ok(());
     }
+
+    /// Add new elements to the head of a list.
+    /// If the key is not present in the DB, and empty list is initialized
+    /// against the key before adding the elements to the head.
+    ///
+    /// # Arguments
+    ///
+    /// * `k` - The key on which list is stored.
+    ///
+    /// * `v` - The values to be added to the head of the list.
+    ///
+    /// # Returns
+    ///
+    /// * `Ok(())` - If values are added successfully to the head of the list.
+    /// * `Err(DBError)` - if key already exists and has non-list data.
+    pub fn lpush(&self, k: String, v: Vec<String>) -> Result<usize, DBError> {
+        let mut data = match self.data.write() {
+            Ok(data) => data,
+            Err(e) => return Err(DBError::Other(format!("{}", e))),
+        };
+
+        let entry = match data.get_mut(k.as_str()) {
+            Some(entry) => Some(entry),
+            None => None,
+        };
+
+        match entry {
+            Some(e) => {
+                let val = &mut e.value;
+                match val {
+                    Value::List(l) => {
+                        for each in v.iter().cloned() {
+                            l.push_front(each);
+                        }
+                        Ok(l.len())
+                    }
+                    _ => Err(DBError::WrongType),
+                }
+            }
+            None => {
+                let list = VecDeque::from(v);
+                let l_len = list.len();
+                data.insert(k.to_string(), Entry::new(Value::List(list)));
+
+                Ok(l_len)
+            }
+        }
+    }
+
+    /// Adds new elements to the tail of a list.
+    /// If the key is not present in the DB, and empty list is initialized
+    /// against the key before adding the elements to the tail.
+    ///
+    /// # Arguments
+    ///
+    /// * `k` - The key on which list is stored.
+    ///
+    /// * `v` - The values to be added to the tail of the list.
+    ///
+    /// # Returns
+    ///
+    /// * `Ok(())` - If value are added successfully to the tail of the list.
+    /// * `Err(DBError)` - if key already exists and has non-list data.
+    pub fn rpush(&self, k: String, v: Vec<String>) -> Result<usize, DBError> {
+        let mut data = match self.data.write() {
+            Ok(data) => data,
+            Err(e) => return Err(DBError::Other(format!("{}", e))),
+        };
+
+        let entry = match data.get_mut(k.as_str()) {
+            Some(entry) => Some(entry),
+            None => None,
+        };
+
+        match entry {
+            Some(e) => {
+                let val = &mut e.value;
+                match val {
+                    Value::List(l) => {
+                        for each in v.iter().cloned() {
+                            l.push_back(each);
+                        }
+                        Ok(l.len())
+                    }
+                    _ => Err(DBError::WrongType),
+                }
+            }
+            None => {
+                let list = VecDeque::from(v);
+                let l_len = list.len();
+                data.insert(k.to_string(), Entry::new(Value::List(list)));
+
+                Ok(l_len)
+            }
+        }
+    }
+
+    /// Returns the specified number of elements of the list stored at key, based on the start and stop indices.
+    /// These offsets can also be negative numbers indicating offsets starting at the end of the list.
+    /// For example, -1 is the last element of the list, -2 the penultimate, and so on.
+    /// Please note that the item at stop index is also included in the result.
+    ///
+    /// If the specified key is not found, an empty list is returned.
+    ///
+    /// # Arguments
+    ///
+    /// * `k` - The key on which list is stored.
+    ///
+    /// * `start_idx` - The start index.
+    ///
+    /// * `stop_idx` - The end index.
+    ///
+    /// # Returns
+    ///
+    /// * `Ok(Vec<String>)` - If values are retrieved successfully from the list.
+    /// * `Err(DBError)` - if key already exists and has non-list data.
+    pub fn lrange(&self, k: String, start_idx: i64, stop_idx: i64) -> Result<Vec<String>, DBError> {
+        let data = match self.data.read() {
+            Ok(data) => data,
+            Err(e) => return Err(DBError::Other(format!("{}", e))),
+        };
+
+        let entry = match data.get(k.as_str()) {
+            Some(entry) => entry,
+            None => return Ok(vec![]),
+        };
+
+        match &entry.value {
+            Value::List(l) => {
+                let l_len = l.len() as i64;
+                let (rounded_start_idx, rounded_stop_idx) =
+                    Self::round_list_indices(l_len, start_idx, stop_idx);
+                Ok(l.range(rounded_start_idx..rounded_stop_idx)
+                    .cloned()
+                    .collect())
+            }
+            _ => Err(DBError::WrongType),
+        }
+    }
+
+    /// Round index to 0, if the given index value is less than zero.
+    /// Round index to list length, if the given index value is greater then the list length.
+    fn round_list_index(list_len: i64, idx: i64) -> usize {
+        if idx < 0 {
+            let idx = list_len - idx.abs();
+            if idx < 0 {
+                return 0;
+            } else {
+                return idx as usize;
+            }
+        }
+
+        if idx >= list_len {
+            return (list_len - 1) as usize;
+        }
+
+        return idx as usize;
+    }
+
+    /// Round the start and stop indices using `Self::round_list_index` method and return them as
+    /// a tuple.
+    /// Special condition: If stop index is lower than start index, return (0, 0).
+    fn round_list_indices(list_len: i64, start_idx: i64, stop_idx: i64) -> (usize, usize) {
+        if stop_idx < start_idx {
+            return (0, 0);
+        }
+
+        let rounded_start_idx = Self::round_list_index(list_len, start_idx);
+        let rounded_stop_idx = Self::round_list_index(list_len, stop_idx);
+
+        if rounded_start_idx < rounded_stop_idx {
+            (rounded_start_idx, rounded_stop_idx + 1)
+        } else if rounded_stop_idx < rounded_start_idx {
+            (0, 0)
+        } else {
+            (rounded_start_idx, rounded_start_idx + 1)
+        }
+    }
 }

 impl Entry {
```

## Handle LPUSH, RPUSH and LRANGE

Implementing these commands is similar to how we did the same for PING, SET and GET. Before proceeding further, have a look at the documentation for the above commands:

* LPUSH - [https://redis.io/docs/latest/commands/lpush/](https://redis.io/docs/latest/commands/lpush/)
* RPUSH - [https://redis.io/docs/latest/commands/rpush/](https://redis.io/docs/latest/commands/rpush/)
* LRANGE - [https://redis.io/docs/latest/commands/lrange/](https://redis.io/docs/latest/commands/lrange/)


If you read the above documentation, you'll see that the LPUSH and RPUSH commands return an integer in RESP format. Currently, our `RespType` doesn't support integer values, so let's add that feature first.

### Add Integer Support in RespType

```diff
diff --git a/src/resp/types.rs b/src/resp/types.rs
index 9dd2930..051a21a 100644
--- a/src/resp/types.rs
+++ b/src/resp/types.rs
@@ -11,6 +11,8 @@ pub enum RespType {
     SimpleError(String),
     /// Refer <https://redis.io/docs/latest/develop/reference/protocol-spec/#arrays>
     Array(Vec<RespType>),
+    /// Refer <https://redis.io/docs/latest/develop/reference/protocol-spec/#integers>
+    Integer(i64),
 }

 use super::RespError;
@@ -139,6 +141,7 @@ impl RespType {
                 Bytes::from_iter(arr_bytes)
             }
             RespType::SimpleError(es) => Bytes::from_iter(format!("-{}\r\n", es).into_bytes()),
+            RespType::Integer(i) => Bytes::from_iter(format!(":{}\r\n", i).into_bytes()),
         };
     }
```

### Add Command Handlers for LPUSH, RPUSH and LRANGE

```rust
// src/command/lpush.rs

use crate::{resp::types::RespType, storage::db::DB};

use super::CommandError;

/// Represents the LPUSH command in Nimblecache.
#[derive(Debug, Clone)]
pub struct LPush {
    key: String,
    values: Vec<String>,
}

impl LPush {
    /// Creates a new `LPUSH` instance from the given arguments.
    ///
    /// # Arguments
    ///
    /// * `args` - A vector of `RespType` representing the arguments to the SET command.
    ///
    /// # Returns
    ///
    /// * `Ok(LPush)` if parsing succeeds.
    /// * `Err(CommandError)` if parsing fails.
    pub fn with_args(args: Vec<RespType>) -> Result<LPush, CommandError> {
        if args.len() < 2 {
            return Err(CommandError::Other(String::from(
                "Wrong number of arguments specified for 'LPUSH' command",
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

        // parse values
        let mut values: Vec<String> = vec![];
        for arg in args[1..].iter() {
            match arg {
                RespType::BulkString(v) => values.push(v.to_string()),
                _ => {
                    return Err(CommandError::Other(String::from(
                        "Invalid argument. Value must be a bulk string",
                    )));
                }
            }
        }

        Ok(LPush {
            key: key.to_string(),
            values,
        })
    }

    /// Executes the LPUSH command.
    ///
    /// # Arguments
    ///
    /// * `db` - The database where the key and values are stored.
    ///
    /// # Returns
    ///
    /// It returns the length of the list if value is successfully written.
    pub fn apply(&self, db: &DB) -> RespType {
        match db.lpush(self.key.clone(), self.values.clone()) {
            Ok(len) => RespType::Integer(len as i64),
            Err(e) => RespType::SimpleError(format!("{}", e)),
        }
    }

    pub fn build_command(&self) -> RespType {
        let mut args: Vec<RespType> = vec![
            RespType::BulkString(String::from("LPUSH")),
            RespType::BulkString(self.key.clone()),
        ];

        let arg_vals = self.values.clone();
        for arg in arg_vals.iter() {
            args.push(RespType::BulkString(arg.to_string()));
        }

        RespType::Array(args)
    }
}
```

```rust
// src/command/rpush.rs

use crate::{resp::types::RespType, storage::db::DB};

use super::CommandError;

/// Represents the RPUSH command in Nimblecache.
#[derive(Debug, Clone)]
pub struct RPush {
    key: String,
    values: Vec<String>,
}

impl RPush {
    /// Creates a new `RPUSH` instance from the given arguments.
    ///
    /// # Arguments
    ///
    /// * `args` - A vector of `RespType` representing the arguments to the SET command.
    ///
    /// # Returns
    ///
    /// * `Ok(RPush)` if parsing succeeds.
    /// * `Err(CommandError)` if parsing fails.
    pub fn with_args(args: Vec<RespType>) -> Result<RPush, CommandError> {
        if args.len() < 2 {
            return Err(CommandError::Other(String::from(
                "Wrong number of arguments specified for 'RPUSH' command",
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

        // parse values
        let mut values: Vec<String> = vec![];
        for arg in args[1..].iter() {
            match arg {
                RespType::BulkString(v) => values.push(v.to_string()),
                _ => {
                    return Err(CommandError::Other(String::from(
                        "Invalid argument. Value must be a bulk string",
                    )));
                }
            }
        }

        Ok(RPush {
            key: key.to_string(),
            values,
        })
    }

    /// Executes the RPUSH command.
    ///
    /// # Arguments
    ///
    /// * `db` - The database where the key and values are stored.
    ///
    /// # Returns
    ///
    /// It returns the length of the list if value is successfully written.
    pub fn apply(&self, db: &DB) -> RespType {
        match db.rpush(self.key.clone(), self.values.clone()) {
            Ok(len) => RespType::Integer(len as i64),
            Err(e) => RespType::SimpleError(format!("{}", e)),
        }
    }

    pub fn build_command(&self) -> RespType {
        let mut args: Vec<RespType> = vec![
            RespType::BulkString(String::from("RPUSH")),
            RespType::BulkString(self.key.clone()),
        ];

        let arg_vals = self.values.clone();
        for arg in arg_vals.iter() {
            args.push(RespType::BulkString(arg.to_string()));
        }

        RespType::Array(args)
    }
}
```

```rust
// src/command/lrange.rs

use crate::{resp::types::RespType, storage::db::DB};

use super::CommandError;

/// Represents the LRANGE command in Nimblecache.
#[derive(Debug, Clone)]
pub struct LRange {
    key: String,
    start_idx: i64,
    end_idx: i64,
}

impl LRange {
    /// Creates a new `LRANGE` instance from the given arguments.
    ///
    /// # Arguments
    ///
    /// * `args` - A vector of `RespType` representing the arguments to the SET command.
    ///
    /// # Returns
    ///
    /// * `Ok(LRange)` if parsing succeeds.
    /// * `Err(CommandError)` if parsing fails.
    pub fn with_args(args: Vec<RespType>) -> Result<LRange, CommandError> {
        if args.len() < 3 {
            return Err(CommandError::Other(String::from(
                "Wrong number of arguments specified for 'LRANGE' command",
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

        // parse start index
        let value = &args[1];
        let start_idx = match value {
            RespType::BulkString(v) => {
                let start_idx = v.parse::<i64>();
                match start_idx {
                    Ok(i) => i,
                    Err(_) => {
                        return Err(CommandError::Other(String::from(
                            "Start index should be an integer",
                        )))
                    }
                }
            }
            _ => {
                return Err(CommandError::Other(String::from(
                    "Invalid argument. Value must be an integer in bulk string format",
                )));
            }
        };

        // parse end index
        let value = &args[2];
        let end_idx = match value {
            RespType::BulkString(v) => {
                let end_idx = v.parse::<i64>();
                match end_idx {
                    Ok(i) => i,
                    Err(_) => {
                        return Err(CommandError::Other(String::from(
                            "End index should be an integer",
                        )))
                    }
                }
            }
            _ => {
                return Err(CommandError::Other(String::from(
                    "Invalid argument. Value must be an integer in bulk string format",
                )));
            }
        };

        Ok(LRange {
            key: key.to_string(),
            start_idx,
            end_idx,
        })
    }

    /// Executes the LRANGE command.
    ///
    /// # Arguments
    ///
    /// * `db` - The database where the key and values are stored.
    ///
    /// # Returns
    ///
    /// It returns the specified number of elements in the list stored at key, based on start and stop indices.
    pub fn apply(&self, db: &DB) -> RespType {
        match db.lrange(self.key.clone(), self.start_idx, self.end_idx) {
            Ok(elems) => {
                let sub_list = elems
                    .iter()
                    .cloned()
                    .map(|e| RespType::BulkString(e))
                    .collect();
                RespType::Array(sub_list)
            }
            Err(e) => RespType::SimpleError(format!("{}", e)),
        }
    }
}
```

```diff
diff --git a/src/command/mod.rs b/src/command/mod.rs
index ba4b8b4..71196fc 100644
--- a/src/command/mod.rs
+++ b/src/command/mod.rs
@@ -3,13 +3,19 @@
 use core::fmt;

 use get::Get;
+use lpush::LPush;
+use lrange::LRange;
 use ping::Ping;
+use rpush::RPush;
 use set::Set;

 use crate::{resp::types::RespType, storage::db::DB};

 mod get;
+mod lpush;
+mod lrange;
 mod ping;
+mod rpush;
 mod set;

 /// Represents the supported Nimblecache commands.
@@ -21,6 +27,12 @@ pub enum Command {
     Set(Set),
     /// The GET command.
     Get(Get),
+    /// The LPUSH command.
+    LPush(LPush),
+    /// The RPUSH command.
+    RPush(RPush),
+    /// The LRANGE command.
+    LRange(LRange),
 }

 impl Command {
@@ -58,6 +70,27 @@ impl Command {
                     Err(e) => return Err(e),
                 }
             }
+            "lpush" => {
+                let cmd = LPush::with_args(Vec::from(args));
+                match cmd {
+                    Ok(cmd) => Command::LPush(cmd),
+                    Err(e) => return Err(e),
+                }
+            }
+            "rpush" => {
+                let cmd = RPush::with_args(Vec::from(args));
+                match cmd {
+                    Ok(cmd) => Command::RPush(cmd),
+                    Err(e) => return Err(e),
+                }
+            }
+            "lrange" => {
+                let cmd = LRange::with_args(Vec::from(args));
+                match cmd {
+                    Ok(cmd) => Command::LRange(cmd),
+                    Err(e) => return Err(e),
+                }
+            }
             _ => {
                 return Err(CommandError::UnknownCommand(ErrUnknownCommand {
                     cmd: cmd_name,
@@ -82,6 +115,9 @@ impl Command {
             Command::Ping(ping) => ping.apply(),
             Command::Set(set) => set.apply(db),
             Command::Get(get) => get.apply(db),
+            Command::LPush(lpush) => lpush.apply(db),
+            Command::RPush(rpush) => rpush.apply(db),
+            Command::LRange(lrange) => lrange.apply(db),
         }
     }
 }
```

## Testing

Start Nimblecache using the below command and use Redis client to connect to it.

```diff
# --port parameter is optional
RUST_LOG=info cargo run -- --port 6380
```

Now, try issuing LPUSH, RPUSH and LRANGE commands to see if they work properly.

## Source code

The source code for this specific part is available at [https://github.com/dheerajgopi/nimblecache/tree/blog-6](https://github.com/dheerajgopi/nimblecache/tree/blog-6).

If you are interested in seeing the git-diff between this part and the previous part of this series, have a look at this commit: [https://github.com/dheerajgopi/nimblecache/commit/4d35dab8ba3ddb4fcfe19149dbac912a506c5903](https://github.com/dheerajgopi/nimblecache/commit/4d35dab8ba3ddb4fcfe19149dbac912a506c5903)

Feel free to check the [main](https://github.com/dheerajgopi/nimblecache/tree/main) branch of the [Nimblecache](https://github.com/dheerajgopi/nimblecache/tree/main) repository to see the latest code.

## Conclusion and what's next

In this article, we successfully implemented the LIST data type in our Redis clone, covering the essential LPUSH, RPUSH, and LRANGE commands.

In the next part of this series, we will implement transactions using the MULTI and EXEC commands. This will allow us to execute multiple commands at once.

Stay tuned!
---
title: "Redis Clone in Rust - Part 2"
subtitle: "Build a Bare-Bones RESP parser"
date: 2024-08-27T00:00:00+05:30
draft: false
tags: ["rust", "redis", "tokio", "database"]
image: /images/blogs/banner-rust-redis-clone-2.png
---

In this article, we explore the basics of RESP (REdis Serialization Protocol), a protocol used by Redis for client-server communication. We begin by understanding the structure and representation of basic RESP data types, such as BulkString, SimpleString, and SimpleError. Next, we build a simple RESP parser that can handle these data types, and then, we modify our existing TCP server to be RESP-aware, allowing it to process RESP-formatted TCP messages. This prepares us for implementing Redis commands and handling more complex RESP types in future articles.

## What is RESP?

RESP (REdis Serialization Protocol) is a binary-safe serialization protocol that supports several data types (Strings, Integers, Arrays, etc.). Redis uses RESP for its client-server communication. The general idea is that the first byte determines the type, and the subsequent bytes constitute the type's content. The type's content might be prefixed by its length so that the parser knows how many bytes to read. The `\r\n` (CRLF) is the protocol's *terminator*, which separates the parts in the type's content.

Here's how a Redis [BulkString](https://redis.io/docs/latest/develop/reference/protocol-spec/#bulk-strings) data type is represented in RESP.

`$<length>\r\n<data>\r\n`

* The dollar sign (`$`) as the first byte indicates it's a BulkString.
* One or more decimal digits (`0`..`9`) as the string's length, in bytes, as an unsigned, base-10 value.
* The CRLF terminator.
* The data.
* A final CRLF.


And here's an example of a BulkString in RESP.

`$5\r\nhello\r\n`

Please note that the RESP format differs for other data types. Redis has provided excellent documentation on RESP and its data types. I highly recommend you go through the documentation before building the RESP parser.

Here's the link to the documentation: [https://redis.io/docs/latest/develop/reference/protocol-spec/](https://redis.io/docs/latest/develop/reference/protocol-spec/)

## Redis data types in Rust

Lets start with a `RespType` enum which will be a wrapper around all supported data types. For now, we will support SimpleString, SimpleError and BulkString.

```rust
// src/resp/types.rs

/// This enum is a wrapper for the different data types in RESP.
#[derive(Clone, Debug)]
pub enum RespType {
    /// Refer <https://redis.io/docs/latest/develop/reference/protocol-spec/#simple-strings>
    SimpleString(String),
    /// Refer <https://redis.io/docs/latest/develop/reference/protocol-spec/#bulk-strings>
    BulkString(String),
    /// Refer <https://redis.io/docs/latest/develop/reference/protocol-spec/#simple-errors>
    SimpleError(String),
}
```

Parsing RESP values from a buffer of raw bytes is pretty straightforward. Before we start with the code for parsing, lets create some Error types which are used in case of parsing failures.

```rust
// src/resp/mod.rs

pub mod types;

/// Represents errors that can occur during RESP parsing.
#[derive(Debug)]
pub enum RespError {
    /// Represents an error in parsing a bulk string, with an error message.
    InvalidBulkString(String),
    /// Represents an error in parsing a simple string, with an error message.
    InvalidSimpleString(String),
    /// Represents any other error with a descriptive message.
    Other(String),
}

impl std::fmt::Display for RespError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            RespError::Other(msg) => msg.as_str().fmt(f),
            RespError::InvalidBulkString(msg) => msg.as_str().fmt(f),
            RespError::InvalidSimpleString(msg) => msg.as_str().fmt(f),
        }
    }
}
```

*Don't forget to add* `mod resp;` *in* `src/main.rs`*, or else the files inside the* `resp` *module wont be considered during compilation process.*

Since we are dealing with buffers of raw bytes, lets add the [bytes](https://crates.io/crates/bytes) crate into the list of dependencies.

```ini
[dependencies]
...
bytes = "1.6.0"
...
```

Let's now start with the actual parsing logic.

### Parsing BulkString

Parsing BulkString involves the following steps:

* Check if the first byte is `$`.
* Read the subsequent bytes until the CRLF terminator (`\r\n`) is encountered. This contains the length (number of bytes) of the actual string data. Once the length is parsed, the parser knows how many bytes to read for extracting the complete string data.
* Read the subsequent bytes based on the length parsed in the previous step. These bytes contain the actual string data.
* The extracted bytes are then converted to a UTF-8 string.


Here's the code for the above logic (Add below code in `src/resp/types.rs` file).

```rust
use bytes::{Bytes, BytesMut};
use super::RespError;

impl RespType {
    /// Parse the given bytes into its respective RESP type and return the parsed RESP value and
    /// the number of bytes read from the buffer.
    ///
    /// More details on the parsing logic is available at
    /// <https://redis.io/docs/latest/develop/reference/protocol-spec/#resp-protocol-description>.
    ///
    /// # Errors
    /// Error will be returned in the following scenarios:
    /// - If first byte is an invalid character.
    /// - If the parsing fails due to encoding issues etc.
    pub fn parse(buffer: BytesMut) -> Result<(RespType, usize)> {
        let c = buffer[0] as char;
        return match c {
            '$' => Self::parse_bulk_string(buffer),
            _ => Err(RespError::Other(String::from(
                "Invalid RESP data type",
            ))),
        };
    }

    /// Parse the given bytes into a BulkString RESP value. This will return the parsed RESP
    /// value and the number of bytes read from the buffer.
    ///
    /// Example BulkString: `$5\r\nhello\r\n`
    ///
    /// # BulkString Parts:
    /// ``
    ///     $      |            5           | \r\n |    hello     | \r\n
    /// identifier | string length in bytes | CRLF | string value | CRLF
    /// ``
    ///
    /// # Parsing Logic:
    /// - The buffer is read until CRLF characters ("\r\n") are encountered.
    /// - That slice of bytes are then parsed into an int. That will be the string length in bytes (let's say `bulkstr_len`)
    /// - `bulkstr_len` number of bytes are read from the buffer again from where it was stopped previously.
    /// - This 2nd slice of bytes is then parsed into an UTF-8 string.
    ///
    /// Note: The first byte in the buffer is skipped since it's just an identifier for the
    /// RESP type and is not the part of the actual value itself.
    pub fn parse_bulk_string(buffer: BytesMut) -> Result<(RespType, usize), RespError> {
        // read until CRLF and parse length
        let (bulkstr_len, bytes_consumed) =
            if let Some((buf_data, len)) = Self::read_till_crlf(&buffer[1..]) {
                let bulkstr_len = Self::parse_usize_from_buf(buf_data)?;
                (bulkstr_len, len + 1)
            } else {
                return Err(RespError::InvalidBulkString(String::from(
                    "Invalid value for bulk string",
                )));
            };

        // validate if buffer contains the complete string data based on
        // the length parsed in the previous step.
        let bulkstr_end_idx = bytes_consumed + bulkstr_len as usize;
        if bulkstr_end_idx >= buffer.len() {
            return Err(RespError::InvalidBulkString(String::from(
                "Invalid value for bulk string length",
            )));
        }

        // convert raw bytes into UTF-8 string.
        let bulkstr = String::from_utf8(buffer[bytes_consumed..bulkstr_end_idx].to_vec());

        match bulkstr {
            Ok(bs) => Ok((RespType::BulkString(bs), bulkstr_end_idx + 2)),
            Err(_) => Err(RespError::InvalidBulkString(String::from(
                "Bulk string value is not a valid UTF-8 string",
            ))),
        }
    }

    // Read the bytes till reaching CRLF ("\r\n")
    fn read_till_crlf(buf: &[u8]) -> Option<(&[u8], usize)> {
        for i in 1..buf.len() {
            if buf[i - 1] == b'\r' && buf[i] == b'\n' {
                return Some((&buf[0..(i - 1)], i + 1));
            }
        }

        None
    }

    // Parse usize from bytes. The number is provided in string format.
    // So convert raw bytes into UTF-8 string and then convert the string
    // into usize.
    fn parse_usize_from_buf(buf: &[u8]) -> Result<usize, RespError> {
        let utf8_str = String::from_utf8(buf.to_vec());
        let parsed_int = match utf8_str {
            Ok(s) => {
                let int = s.parse::<usize>();
                match int {
                    Ok(n) => Ok(n),
                    Err(_) => Err(RespError::Other(String::from(
                        "Invalid value for an integer",
                    ))),
                }
            }
            Err(_) => Err(RespError::Other(String::from("Invalid UTF-8 string"))),
        };

        parsed_int
    }
}
```

### Parsing SimpleString

Parsing SimpleString is even simpler. It does not have the length prefix as found in BulkString. The logic is as follows:

* Check if the first byte is `+`.
* Read the subsequent bytes until the CRLF terminator (`\r\n`) is encountered. These bytes contain the actual string data.
* The extracted bytes are then converted to a UTF-8 string.


Here's the code for the above logic (Modify `src/resp/types.rs` with the code given below).

```rust
impl RespType {

    ...
    ...

    pub fn parse(buffer: BytesMut) -> Result<(RespType, usize), RespError> {
        ...
        ...
        return match c {
            '$' => Self::parse_bulk_string(buffer),
            '+' => Self::parse_simple_string(buffer),
            _ => Err(RespError::Other(String::from(
                "Invalid RESP data type",
            ))),
        };
    }

    ...
    ...

    /// Parse the given bytes into a SimpleString RESP value. This will return the parsed RESP
    /// value and the number of bytes read from the buffer.
    ///
    /// Example SimpleString: `+OK\r\n`
    ///
    /// # SimpleString Parts:
    /// ``
    ///      +      |      OK      | \r\n
    ///  identifier | string value | CRLF
    /// ``
    ///
    /// # Parsing Logic:
    /// - The buffer is read until CRLF characters ("\r\n") are encountered. That slice of bytes are then
    /// parsed into an UTF-8 string.
    pub fn parse_simple_string(buffer: BytesMut) -> Result<(RespType, usize), RespError> {
        // read until CRLF and parse the bytes into an UTF-8 string.
        if let Some((buf_data, len)) = Self::read_till_crlf(&buffer[1..]) {
            let utf8_str = String::from_utf8(buf_data.to_vec());

            return match utf8_str {
                Ok(simple_str) => Ok((RespType::SimpleString(simple_str), len + 1)),
                Err(_) => {
                    return Err(RespError::InvalidSimpleString(String::from(
                        "Simple string value is not a valid UTF-8 string",
                    )))
                }
            };
        }

        Err(RespError::InvalidSimpleString(String::from(
            "Invalid value for simple string",
        )))
    }

    ...
    ...
}
```

Parsing SimpleError is the same as parsing SimpleString, except the first byte should be `-` instead of `+`. However, a separate parsing function for SimpleError is not needed right now because errors are usually sent back to the client as a response and are not part of the request message.

## RESP-aware TCP server loop

Now that we have a basic RESP parser, let's convert our existing TCP server into a RESP-aware echo server.

For this we need to make modifications on `src/resp/types.rs` and `/src/server.rs`.

Below are the modifications made to `src/resp/types.rs`. This is for adding a method which converts the RESP type back into raw bytes.

```rust
impl RespType {
    ...
    ...

    /// Convert the RESP value into its byte values.
    pub fn to_bytes(&self) -> Bytes {
        return match self {
            RespType::SimpleString(ss) => Bytes::from_iter(format!("+{}\r\n", ss).into_bytes()),
            RespType::BulkString(bs) => {
                let bulkstr_bytes = format!("${}\r\n{}\r\n", bs.chars().count(), bs).into_bytes();
                Bytes::from_iter(bulkstr_bytes)
            }
            RespType::SimpleError(es) => Bytes::from_iter(format!("-{}\r\n", es).into_bytes()),
        };
    }

    ...
    ...
}
```

The modifications made to `/src/server.rs` are as given below.

```diff
diff --git a/src/server.rs b/src/server.rs
index ae3fef9..a249175 100644
--- a/src/server.rs
+++ b/src/server.rs
@@ -3,15 +3,18 @@
 // anyhow provides the Error and Result types for convenient error handling
 use anyhow::{Error, Result};

+use bytes::BytesMut;
 // log crate provides macros for logging at various levels (error, warn, info, debug, trace)
 use log::error;

 use tokio::{
     // AsyncWriteExt trait provides asynchronous write methods like write_all
-    io::AsyncWriteExt,
+    io::{AsyncReadExt, AsyncWriteExt},
     net::{TcpListener, TcpStream},
 };

+use crate::resp::types::RespType;
+
 /// The Server struct holds the tokio TcpListener which listens for
 /// incoming TCP connections.
 #[derive(Debug)]
@@ -44,8 +47,21 @@ impl Server {
             // Spawn a new asynchronous task to handle the connection.
             // This allows the server to handle multiple connections concurrently.
             tokio::spawn(async move {
-                // Write a "Hello!" message to the client.
-                if let Err(e) = &mut sock.write_all("Hello!".as_bytes()).await {
+                // read the TCP message and move the raw bytes into a buffer
+                let mut buffer = BytesMut::with_capacity(512);
+                if let Err(e) = sock.read_buf(&mut buffer).await {
+                    panic!("Error reading request: {}", e);
+                }
+
+                // Try parsing the RESP data from the bytes in the buffer.
+                // If parsing fails return the error message as a RESP SimpleError data type.
+                let resp_data = match RespType::parse(buffer) {
+                    Ok((data, _)) => data,
+                    Err(e) => RespType::SimpleError(format!("{}", e)),
+                };
+
+                // Echo the RESP message back to the client.
+                if let Err(e) = &mut sock.write_all(&resp_data.to_bytes()[..]).await {
                     // Log the error and panic if there is an issue writing the response.
                     error!("{}", e);
                     panic!("Error writing response")
@@ -70,4 +86,4 @@ impl Server {
             }
         }
     }
-}
\ No newline at end of file
+}
```

## Running the TCP server

Run the following command to run the application.

```bash
RUST_LOG=info cargo run
```

`RUST_LOG=info` will set the log level to `info`.

Use the below commands to send RESP data using `nc`.

```bash
# send bulk string. This will echo the same bulk string back.
{ echo -e '$5\r\nhello\r\n'; sleep 1; } | nc localhost 6379

# send simple string. This will echo the same bulk string back.
{ echo -e '+OK\r\n'; sleep 1; } | nc localhost 6379

# send invalid resp data. This will respond with a simple error.
{ echo -e '+OK'; sleep 1; } | nc localhost 6379
```

## Source Code

The source code for this specific part is available at [https://github.com/dheerajgopi/nimblecache/tree/blog-2](https://github.com/dheerajgopi/nimblecache/tree/blog-2).

If you are interested in seeing the git-diff between this part and the previous part of this series, have a look at this commit: [https://github.com/dheerajgopi/nimblecache/commit/30a65f7bb446e875e7b340a19c9fa50f2f779a17](https://github.com/dheerajgopi/nimblecache/commit/30a65f7bb446e875e7b340a19c9fa50f2f779a17)

Feel free to check the [main](https://github.com/dheerajgopi/nimblecache/tree/main) branch of the Nimblecache repository to see the latest code.

## Conclusion and what's next

In this article, we explored the basics of RESP and built a simple RESP parser that can handle RESP data types like BulkString, SimpleString and SimpleError. We also modified our TCP server to be RESP-aware, allowing it to process RESP-formatted messages.

In the next part of this series, we will delve into implementing Redis command parsing and handling more RESP types. Stay tuned!
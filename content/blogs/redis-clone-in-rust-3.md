---
title: "Redis Clone in Rust - Part 3"
subtitle: "Parsing Redis Commands with tokio-util"
date: 2024-08-28T00:00:00+05:30
draft: false
tags: ["rust", "redis", "tokio", "database"]
image: /images/blogs/banner-rust-redis-clone-3.png
---

In our ongoing series on building a Redis clone using Rust, we've covered the basics of setting up a TCP server and parsing basic RESP data types. Now, it's time to advance our project by understanding the structure of Redis commands and how to parse them efficiently.

In this article, we'll explore how Redis commands are structured and sent to the server, learn to parse incoming TCP messages as Redis commands using the [`Framed`](https://docs.rs/tokio-util/latest/tokio_util/codec/struct.Framed.html) utility from the [tokio-util](https://crates.io/crates/tokio-util) library, and understand the RESP Array type as part of this process.

## Structure of a Redis command

A client sends a request to the Redis server as an array of strings, where the first element is the command name, followed by its arguments. In RESP terms, a Redis command can be represented by an Array of BulkStrings. At this point, our RESP parser supports only SimpleString, Bulk String and SimpleError. Now its time to add support for RESP array.

### RESP Array

Here's how an [Array](https://redis.io/docs/latest/develop/reference/protocol-spec/#arrays) data type is represented in RESP.

`*<number-of-elements>\r\n<element-1>...<element-n>`

* The asterisk symbol (`*`) as the first byte indicates it's an Array.
* One or more decimal digits (`0`..`9`) as the string's length, in bytes, as an unsigned, base-10 value.
* The CRLF terminator.
* An additional RESP type for every element of the array.

Here's an example for a RESP array which contains 2 BulkStrings:

`*2\r\n$4\r\nPING\r\n$5\r\nhello\r\n`,

which represents the Redis PING command in RESP format.

Now, let's add support for RESP Arrays in our Redis clone. Please note that the functions `parse_array_len` and `parse_bulk_string_len` in the code below will be used later in this article. You don't need to worry about the intent of those functions at this point :)

```diff
diff --git a/src/resp/types.rs b/src/resp/types.rs
index cae0dcb..ebedb2c 100644
--- a/src/resp/types.rs
+++ b/src/resp/types.rs
@@ -7,6 +7,8 @@ pub enum RespType {
     BulkString(String),
     /// Refer <https://redis.io/docs/latest/develop/reference/protocol-spec/#simple-errors>
     SimpleError(String),
+    /// Refer <https://redis.io/docs/latest/develop/reference/protocol-spec/#arrays>
+    Array(Vec<RespType>),
 }

 use super::RespError;
@@ -125,10 +127,87 @@ impl RespType {
                 let bulkstr_bytes = format!("${}\r\n{}\r\n", bs.chars().count(), bs).into_bytes();
                 Bytes::from_iter(bulkstr_bytes)
             }
+            RespType::Array(arr) => {
+                let mut arr_bytes = format!("*{}\r\n", arr.len()).into_bytes();
+                arr.iter()
+                    .map(|v| v.to_bytes())
+                    .for_each(|b| arr_bytes.extend(b));
+
+                Bytes::from_iter(arr_bytes)
+            }
             RespType::SimpleError(es) => Bytes::from_iter(format!("-{}\r\n", es).into_bytes()),
         };
     }

+    /// Parses the length of a RESP array from the given byte buffer.
+    ///
+    /// This function attempts to read the first few bytes of a RESP array to determine its length.
+    /// It expects the input to start with a '*' character followed by the length and terminated by CRLF.
+    ///
+    /// # Arguments
+    ///
+    /// * `src` - A `BytesMut` containing the bytes to parse.
+    ///
+    /// # Returns
+    ///
+    /// * `Ok(Some((usize, usize)))` - If successful, returns a tuple containing:
+    ///   - The parsed length of the array
+    ///   - The number of bytes read from the input
+    /// * `Ok(None)` - If there's not enough data in the buffer to parse the length
+    /// * `Err(RespError)` - If the input is not a valid RESP array prefix or if parsing fails
+    pub fn parse_array_len(src: BytesMut) -> Result<Option<(usize, usize)>, RespError> {
+        let (array_prefix_bytes, bytes_read) = match Self::read_till_crlf(&src[..]) {
+            Some((b, size)) => (b, size),
+            None => return Ok(None),
+        };
+
+        if bytes_read < 4 || array_prefix_bytes[0] as char != '*' {
+            return Err(RespError::InvalidArray(String::from(
+                "Not a valid RESP array",
+            )));
+        }
+
+        match Self::parse_usize_from_buf(&array_prefix_bytes[1..]) {
+            Ok(len) => Ok(Some((len, bytes_read))),
+            Err(e) => Err(e),
+        }
+    }
+
+    /// Parses the length of a RESP bulk string from the given byte buffer.
+    ///
+    /// This function attempts to read the first few bytes of a RESP bulk string to determine its length.
+    /// It expects the input to start with a '$' character followed by the length and terminated by CRLF.
+    ///
+    /// # Arguments
+    ///
+    /// * `src` - A `BytesMut` containing the bytes to parse.
+    ///
+    /// # Returns
+    ///
+    /// * `Ok(Some((usize, usize)))` - If successful, returns a tuple containing:
+    ///   - The parsed length of the bulk string
+    ///   - The number of bytes read from the input
+    /// * `Ok(None)` - If there's not enough data in the buffer to parse the length
+    /// * `Err(RespError)` - If the input is not a valid RESP bulk string prefix or if parsing fails
+    ///
+    pub fn parse_bulk_string_len(src: BytesMut) -> Result<Option<(usize, usize)>, RespError> {
+        let (bulkstr_prefix_bytes, bytes_read) = match Self::read_till_crlf(&src[..]) {
+            Some((b, size)) => (b, size),
+            None => return Ok(None),
+        };
+
+        if bytes_read < 4 || bulkstr_prefix_bytes[0] as char != '$' {
+            return Err(RespError::InvalidBulkString(String::from(
+                "Not a valid RESP bulk string",
+            )));
+        }
+
+        match Self::parse_usize_from_buf(&bulkstr_prefix_bytes[1..]) {
+            Ok(len) => Ok(Some((len, bytes_read))),
+            Err(e) => Err(e),
+        }
+    }
+
     // Read the bytes till reaching CRLF ("\r\n")
     fn read_till_crlf(buf: &[u8]) -> Option<(&[u8], usize)> {
         for i in 1..buf.len() {
```

```diff
diff --git a/src/resp/mod.rs b/src/resp/mod.rs
index 9cdde64..24a806d 100644
--- a/src/resp/mod.rs
+++ b/src/resp/mod.rs
@@ -1,3 +1,4 @@
+pub mod frame;
 pub mod types;

 /// Represents errors that can occur during RESP parsing.
@@ -7,6 +8,8 @@ pub enum RespError {
     InvalidBulkString(String),
     /// Represents an error in parsing a simple string, with an error message.
     InvalidSimpleString(String),
+    /// Represents an error in parsing an array, with an error message.
+    InvalidArray(String),
     /// Represents any other error with a descriptive message.
     Other(String),
 }
@@ -17,6 +20,7 @@ impl std::fmt::Display for RespError {
             RespError::Other(msg) => msg.as_str().fmt(f),
             RespError::InvalidBulkString(msg) => msg.as_str().fmt(f),
             RespError::InvalidSimpleString(msg) => msg.as_str().fmt(f),
+            RespError::InvalidArray(msg) => msg.as_str().fmt(f),
         }
     }
 }
```

## Parsing Redis commands using tokio-util Framed

Before we handle Redis command parsing, it's important to revisit the existing TCP message parsing logic our TCP server. Specifically, the below lines:

```rust
// src/server.rs

...
...

// read the TCP message and move the raw bytes into a buffer
let mut buffer = BytesMut::with_capacity(512);
if let Err(e) = sock.read_buf(&mut buffer).await {
    panic!("Error reading request: {}", e);
}

// Try parsing the RESP data from the bytes in the buffer.
// If parsing fails return the error message as a RESP SimpleError data type.
let resp_data = match RespType::parse(buffer) {
    Ok((data, _)) => data,
    Err(e) => RespType::SimpleError(format!("{}", e)),
};

...
...
```

Some of the issues with the above code are:

* Potential for incomplete reads: The code reads into the buffer once, but doesn't account for partial reads where the full command might not have arrived yet.
* Conversely, if multiple commands arrive in one read, the code might not handle it correctly.
* Crude error handling: There's no proper error handling. The code just panics if it fails to read into the buffer.

The root cause of these issues is that we are using a buffer with a fixed size. While it is possible to fix these problems, manually managing the buffer can be error-prone. In such cases, it is better to use existing utilities like [`Framed`](https://docs.rs/tokio-util/latest/tokio_util/codec/struct.Framed.html) in [tokio-util](https://crates.io/crates/tokio-util).

### What is tokio-util Framed?

To put it simply, the `Framed` utility is a tool for converting a blob of bytes into a domain-specific type and vice versa. In our case, `Framed` is going to operate on the socket, converting the raw bytes of the incoming TCP messages into `RespType` values, and also converting `RespType` values back into raw bytes for outgoing TCP messages. The best part is that `Framed` handles the buffering internally.

`Framed` relies on two simple traits: `Decode` and `Encode`. Together, these form a `Codec`, which is necessary for `Framed` to work on a socket. So, as part of implementing `Framed` this is what we need to do:

* Implement `Decoder` trait which will convert the incoming TCP messages into `Vec<RespType>`.
* Implement `Encoder` trait which will convert `RespType` values into raw bytes which will be sent via the outgoing TCP message.

Before implementing the `Codec`, let's add the `tokio-util` dependency in `Cargo.toml`.

```diff
diff --git a/Cargo.toml b/Cargo.toml
index 0e36917..b50bb57 100644
--- a/Cargo.toml
+++ b/Cargo.toml
@@ -15,5 +15,7 @@ tokio = { version = "1.38.0", features = [
     "io-util",
     "sync",
 ] }
+tokio-util = { version = "0.7.11", features = ["codec"] }
 log = "0.4.22"
 env_logger = "0.11.3"
+futures = { version = "0.3", default-features = true }
```

### Implementing Decoder

The job of the decoder is to look into a data buffer and determine if it contains enough data to produce a complete item (`Vec<RespType>`) in the stream.

A decoder should work based on the below logic:

* If the buffer contains enough data to construct the Redis command (`Vec<RespType>`), the decoder should remove those bytes from the buffer and return `Ok(Some(Vec<RespType>))`.
* If the buffer does not contain enough data to construct the Redis command, leave the buffer unmodified and return `Ok(None)`. The `Framed` utility will then read more data from the TCP stream and append those bytes to the buffer. After that, the decoder is called again to check if it has enough data to produce the command.
* In case of any decoding errors, return `Err(std::io::Error)`.

The code for the decoder is given below. Here are few points to notice:

* The struct `RespCommandFrame`, implements the `Decoder` trait. It utilizes a `CommandBuilder` struct internally to accumulate the parts of a command.
* The decoder examines the first few bytes to find the RESP array length. If it finds the length, it initializes the `CommandBuilder` struct. If the array length is `n`, the decoder expects `n` BulkStrings as part of the incoming TCP message.
* The decoder then tries to read `n` BulkStrings and add each item to the `CommandBuilder`. Once all `n` items are added to the `CommandBuilder`, the command (`Vec<RespType>`) is returned successfully.
* The functions `RespType::parse_array_len` and `RespType::parse_bulk_string_len`, mentioned earlier in this article, are utilized in the decoder.

```rust
// src/resp/frame.rs

use bytes::{Buf, BufMut};
use core::fmt;
use std::io::Error;
use tokio_util::codec::{Decoder, Encoder};

use crate::resp::types::RespType;

use super::RespError;

/// This codec handles Nimblecache commands, which are always represented
/// as array of bulk strings in the RESP (REdis Serialization Protocol) protocol.
///
/// The codec uses a `CommandBuilder` internally to construct the array of bulk strings
/// that make up a Nimblecache command.
pub struct RespCommandFrame {
    /// Builder for appending the bulk strings in the command array.
    cmd_builder: Option<CommandBuilder>,
}

impl RespCommandFrame {
    /// Creates a new `RespCommandFrame`.
    ///
    /// # Returns
    ///
    /// A new instance of `RespCommandFrame` with no command builder initialized.
    pub fn new() -> RespCommandFrame {
        RespCommandFrame { cmd_builder: None }
    }
}

impl Decoder for RespCommandFrame {
    type Item = Vec<RespType>;

    type Error = std::io::Error;

    /// Decodes bytes from the input stream into a `Vec<RespType>` representing a Nimblecache command.
    ///
    /// This method implements the RESP protocol decoding logic, specifically handling
    /// arrays of bulk strings which represent Nimblecache commands. It uses a `CommandBuilder`
    /// to accumulate the parts of the command as they are received.
    ///
    /// # Arguments
    ///
    /// * `src` - A mutable reference to the input buffer containing bytes to decode.
    ///
    /// # Returns
    ///
    /// * `Ok(Some(Vec<RespType>))` if a complete command (array of bulk strings) was successfully decoded.
    /// * `Ok(None)` if more data is needed to complete the command.
    /// * `Err(std::io::Error)` if an error occurred during decoding.
    fn decode(
        &mut self,
        src: &mut bytes::BytesMut,
    ) -> std::result::Result<Option<Self::Item>, Self::Error> {
        // A command in RESP protocol should always be an array of Bulk Strings.
        // Check the first 2 bytes to validate if its a RESP array.
        if self.cmd_builder.is_none() {
            let (cmd_len, bytes_read) = match RespType::parse_array_len(src.clone()) {
                Ok(arr_len) => match arr_len {
                    Some((len, bytes_read)) => (len, bytes_read),
                    None => return Ok(None),
                },
                Err(e) => {
                    return Err(Error::new(
                        std::io::ErrorKind::InvalidData,
                        FrameError::from(e),
                    ));
                }
            };

            // initilize command builder, if its a valid RESP array.
            self.cmd_builder = Some(CommandBuilder::new(cmd_len));

            // advance buffer
            src.advance(bytes_read);
        }

        // Read all bytes in buffer
        while src.len() != 0 {
            // Validate and check the length of next bulk string
            let (bulkstr_len, bytes_read) = match RespType::parse_bulk_string_len(src.clone()) {
                Ok(bulkstr_len) => match bulkstr_len {
                    Some((len, bytes_read)) => (len, bytes_read),
                    None => return Ok(None),
                },
                Err(e) => {
                    return Err(Error::new(
                        std::io::ErrorKind::InvalidData,
                        FrameError::from(e),
                    ));
                }
            };

            // A bulk string has the below format
            //
            // `${string length in bytes }\r\n{string value}\r\n`
            //
            // Check if the buffer contains the required number of bytes to parse
            // the bulk string (including the CRLF at the end)
            let bulkstr_bytes = bulkstr_len + bytes_read + 2;
            if src.len() < bulkstr_bytes {
                return Ok(None);
            }

            // now that its sure the buffer has all the bytes required to parse the bulk string, parse it.
            let (bulkstr, bytes_read) = match RespType::parse_bulk_string(src.clone()) {
                Ok((resp_type, bytes_read)) => (resp_type, bytes_read),
                Err(e) => {
                    return Err(Error::new(
                        std::io::ErrorKind::InvalidData,
                        FrameError::from(e),
                    ));
                }
            };

            // append the bulk string to the command builder
            self.cmd_builder.as_mut().unwrap().add_part(bulkstr);

            // advance buffer
            src.advance(bytes_read);

            // if the command builder has all the parts, return it, else check buffer again
            let cmd_builder = self.cmd_builder.as_ref().unwrap();
            if cmd_builder.all_parts_received() {
                let cmd = cmd_builder.build();
                self.cmd_builder = None;
                return Ok(Some(cmd));
            }
        }

        Ok(None)
    }
}

/// This struct is used to accumulate the parts of a Nimblecache command, which are
/// typically represented as an array of bulk strings in the RESP protocol.
struct CommandBuilder {
    parts: Vec<RespType>,
    num_parts: usize,
    parts_parsed: usize,
}

impl CommandBuilder {
    /// Creates a new `CommandBuilder` with the specified number of parts.
    pub fn new(num_parts: usize) -> CommandBuilder {
        CommandBuilder {
            parts: vec![],
            num_parts,
            parts_parsed: 0,
        }
    }

    /// Adds a part to the command being built and increments the count of parsed parts.
    ///
    /// # Arguments
    ///
    /// * `part` - A `RespType` representing a part of the command.
    pub fn add_part(&mut self, part: RespType) {
        self.parts.push(part);
        self.parts_parsed += 1;
    }

    /// Checks if all expected parts of the command have been received.
    ///
    /// # Returns
    ///
    /// `true` if the number of parsed parts equals the expected number of parts,
    /// `false` otherwise.
    pub fn all_parts_received(&self) -> bool {
        self.num_parts == self.parts_parsed
    }

    /// Builds and returns the complete command as a vector of RESP values.
    ///
    /// # Returns
    ///
    /// A vector of `RespType` containing all the parts of the command.
    pub fn build(&self) -> Vec<RespType> {
        self.parts.clone()
    }
}

/// Represents error that can occur during RESP command frame parsing.
#[derive(Debug)]
pub struct FrameError {
    err: RespError,
}

impl FrameError {
    pub fn from(err: RespError) -> FrameError {
        FrameError { err }
    }
}

impl std::error::Error for FrameError {}

impl fmt::Display for FrameError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        self.err.fmt(f)
    }
}
```

### Implementing Encoder

The job of the encoder is to convert `RespType` values into raw bytes and write the same into the buffer.

Compared to the decoder, this is straight-forward to implement. Add the below code to `src/resp/frame.rs` for implementing `Encoder` trait.

```rust
impl Encoder<RespType> for RespCommandFrame {
    type Error = std::io::Error;

    /// Encodes a `RespType` into bytes and writes them to the output buffer.
    ///
    /// It's primarily used for sending responses to Nimblecache commands.
    ///
    /// # Arguments
    ///
    /// * `item` - The `RespType` to encode.
    /// * `dst` - The output buffer to write the encoded bytes to.
    ///
    /// # Returns
    ///
    /// * `Ok(())` if the encoding was successful.
    /// * `Err(std::io::Error)` if an error occurred during encoding.
    fn encode(&mut self, item: RespType, dst: &mut bytes::BytesMut) -> Result<(), Self::Error> {
        dst.put_slice(&item.to_bytes());

        Ok(())
    }
}
```

### Using Framed

Now let's use `Framed` with `RespCommandFrame` codec in our server loop. Make the modifications to the code as given below:

```rust
// src/handler.rs

use anyhow::Result;
use futures::{SinkExt, StreamExt};
use log::error;
use tokio::net::TcpStream;
use tokio_util::codec::Framed;

use crate::resp::{frame::RespCommandFrame, types::RespType};

/// Handles RESP command frames over a single TCP connection.
pub struct FrameHandler {
    /// The framed connection using `RespCommandFrame` as the codec.
    conn: Framed<TcpStream, RespCommandFrame>,
}
impl FrameHandler {
    /// Creates a new `FrameHandler` instance.
    pub fn new(conn: Framed<TcpStream, RespCommandFrame>) -> FrameHandler {
        FrameHandler { conn }
    }

    /// Handles incoming RESP command frames.
    ///
    /// This method continuously reads command frames from the connection,
    /// and echo it back to the client. It continues until
    /// an error occurs or the connection is closed.
    ///
    /// # Returns
    ///
    /// A `Result` indicating whether the operation succeeded or failed.
    ///
    /// # Errors
    ///
    /// This method will return an error if there's an issue with reading
    /// from or writing to the connection.
    pub async fn handle(mut self) -> Result<()> {
        while let Some(resp_cmd) = self.conn.next().await {
            match resp_cmd {
                Ok(cmd_frame) => {
                    // Write the RESP response into the TCP stream.
                    if let Err(e) = self.conn.send(RespType::Array(cmd_frame)).await {
                        error!("Error sending response: {}", e);
                        break;
                    }
                }
                Err(e) => {
                    error!("Error reading the request: {}", e);
                    break;
                }
            };

            // flush the buffer into the TCP stream.
            self.conn.flush().await?;
        }

        Ok(())
    }
}
```

```diff
diff --git a/src/server.rs b/src/server.rs
index a249175..9e45bee 100644
--- a/src/server.rs
+++ b/src/server.rs
@@ -3,17 +3,13 @@
 // anyhow provides the Error and Result types for convenient error handling
 use anyhow::{Error, Result};

-use bytes::BytesMut;
 // log crate provides macros for logging at various levels (error, warn, info, debug, trace)
 use log::error;

-use tokio::{
-    // AsyncWriteExt trait provides asynchronous write methods like write_all
-    io::{AsyncReadExt, AsyncWriteExt},
-    net::{TcpListener, TcpStream},
-};
+use tokio::net::{TcpListener, TcpStream};
+use tokio_util::codec::Framed;

-use crate::resp::types::RespType;
+use crate::{handler::FrameHandler, resp::frame::RespCommandFrame};

 /// The Server struct holds the tokio TcpListener which listens for
 /// incoming TCP connections.
@@ -35,7 +31,7 @@ impl Server {
             // accept a new TCP connection.
             // If successful the corresponding TcpStream is stored
             // in the variable `sock`, else a panic will occur.
-            let mut sock = match self.accept_conn().await {
+            let sock = match self.accept_conn().await {
                 Ok(stream) => stream,
                 // Log the error and panic if there is an issue accepting a connection.
                 Err(e) => {
@@ -44,29 +40,17 @@ impl Server {
                 }
             };

+            // Use RespCommandFrame codec to read incoming TCP messages as Redis command frames,
+            // and to write RespType values into outgoing TCP messages.
+            let resp_command_frame = Framed::with_capacity(sock, RespCommandFrame::new(), 8 * 1024);
+
             // Spawn a new asynchronous task to handle the connection.
             // This allows the server to handle multiple connections concurrently.
             tokio::spawn(async move {
-                // read the TCP message and move the raw bytes into a buffer
-                let mut buffer = BytesMut::with_capacity(512);
-                if let Err(e) = sock.read_buf(&mut buffer).await {
-                    panic!("Error reading request: {}", e);
-                }
-
-                // Try parsing the RESP data from the bytes in the buffer.
-                // If parsing fails return the error message as a RESP SimpleError data type.
-                let resp_data = match RespType::parse(buffer) {
-                    Ok((data, _)) => data,
-                    Err(e) => RespType::SimpleError(format!("{}", e)),
-                };
-
-                // Echo the RESP message back to the client.
-                if let Err(e) = &mut sock.write_all(&resp_data.to_bytes()[..]).await {
-                    // Log the error and panic if there is an issue writing the response.
-                    error!("{}", e);
-                    panic!("Error writing response")
+                let handler = FrameHandler::new(resp_command_frame);
+                if let Err(e) = handler.handle().await {
+                    error!("Failed to handle command: {}", e);
                 }
-                // The connection is closed automatically when `sock` goes out of scope.
             });
         }
     }
```

```diff
diff --git a/src/main.rs b/src/main.rs
index 7b13f63..dee666f 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -1,6 +1,7 @@
 // src/main.rs

 // Include the server module defined in server.rs
+mod handler;
 mod resp;
 mod server;
```

## Running the TCP server

Run the following command to run the application.

```bash
RUST_LOG=info cargo run
```

`RUST_LOG=info` will set the log level to `info`.

Use the below command to send Redis command using `nc`.

```bash
# send a RESP array. This will echo the same bulk string back.
{ echo -e '*2\r\n$4\r\nPING\r\n$5\r\nhello\r\n'; sleep 1; } | nc localhost 6379
```

With the new changes, you might notice that our server won't drop the TCP connection after sending the response back to the client. This is a good thing. However, it also means that testing our Redis clone using `nc` will become even more clunkier. We'll try to eliminate `nc` and use an actual Redis client in the following parts.

## Source Code

The source code for this specific part is available at [https://github.com/dheerajgopi/nimblecache/tree/blog-3](https://github.com/dheerajgopi/nimblecache/tree/blog-3).

If you are interested in seeing the git-diff between this part and the previous part of this series, have a look at this commit: [https://github.com/dheerajgopi/nimblecache/commit/907fa068e33296df53696c36f539879fd45d29a2](https://github.com/dheerajgopi/nimblecache/commit/907fa068e33296df53696c36f539879fd45d29a2)

Feel free to check the [main](https://github.com/dheerajgopi/nimblecache/tree/main) branch of the [Nimblecache](https://github.com/dheerajgopi/nimblecache/tree/main) repository to see the latest code.

## Conclusion and what's next

In this article, we've learned about the structure of Redis commands and how to parse them efficiently using the `Framed` utility from the `tokio-util` library. It's worth noting that `Framed` can be applied in any server application which involves parsing and writing of protocol-specific messages.

In the next part of this series, we will focus on handling a real Redis command. Stay tuned!
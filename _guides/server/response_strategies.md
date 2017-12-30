---
title: Response Strategies
---

## Overview

The [Echo, echo, echo](./echo) guide discusses how to transform
request bodies into response bodies, and the [Handling
Posts](./handle_post) guide discusses processing more complex request
bodies. We will now look into more advanced ways of generating
responses.

Two basic approaches are examined. The first handles blocking i/o or
long processing times. These require a separate thread and the
`futures` crate's channels. We will discuss the threaded approach in
the context of serving files to a client, but this is readily adapted
to database access and/or complex page rendering. A working example of
file serving is included in the hyper distribution at
[send_file.rs](https://github.com/hyperium/hyper/blob/master/examples/send_file.rs).

The second approach illustrates how to use `tokio` aware i/o. We will
illustrate it by accessing a simple web based api. In some ways this
is simpler than the threaded approach, but the setup of our server is
more complex. We will need access to the underlying `tokio` engine to
handle our queries. A working example of web api access is included in
the hyper distribution at
[web_api.rs](https://github.com/hyperium/hyper/blob/master/examples/web_api.rs).

We start with extracting the query information from the request (which
can be from form parameters, cookies, headers, etc, but this guide
will simply use the request path). This information is used to create
the response future (`Service::Future`, which is `Box<Future<Item =
Self::Response, Error = Self::Error>>` here). Finally, the response
body stream is assigned.

All the operations that can affect the response status code need to
take place in the response future, not the response body stream. The
response body stream cannot change the status of the response. For
example, if that stream tries to open a file that does not exist, it
cannot return a 404. The file needs to be opened before the body
stream is created and the handle passed to the stream.

## File Serving

At the time of this writing, there is no cross platform way of
accessing the filesystem with futures. We will use blocking I/O in a
separate thread. This generalizes to anything that needs to run in a
separate thread, such as database access or long computations.

### Simple File Serving

We will start with a simple approach that reads the entire file into a
buffer. This is only appropriate for small quanities of data, such as
a single data base row, or situations where all the long processing
take place before the buffer is generated.

```rust
fn simple_file_send(f: &str) -> Box<Future<Item = Response, Error = hyper::Error>> {
    let filename = f.to_string(); // we need to copy for lifetime issues
    let (tx, rx) = oneshot::channel();
```

For small files, we can do all the work with a single
`futures::sync::oneshot::channel` to communicate with our spawned
thread:


```rust
    thread::spawn(move || {
        let mut file = match File::open(filename) {
            Ok(f) => f,
            Err(_) => {
                tx.send(Response::new()
                        .with_status(StatusCode::NotFound)
                        .with_header(ContentLength(NOTFOUND.len() as u64))
                        .with_body(NOTFOUND))
                    .expect("Send error on open");
                return;
            },
        };
```

First, we attempt to open the file, returning a 404 if the file does
not exists.

```rust		
        let mut buf: Vec<u8> = Vec::new();
        match copy(&mut file, &mut buf) {
            Ok(_) => {
                let res = Response::new()
                    .with_header(ContentLength(buf.len() as u64))
                    .with_body(buf);
                tx.send(res).expect("Send error on successful file read");
            },
            Err(_) => {
                tx.send(Response::new().with_status(StatusCode::InternalServerError)).
                    expect("Send error on error reading file");
            },
        };
    });
```

Next we read whole file into a `Vec<u8>` buffer. We send a Response
with the buffer as the body out the transmit channel. On an error, we
instead return an Internal Service Error.

```rust		
    Box::new(rx.map_err(|e| Error::from(io::Error::new(io::ErrorKind::Other, e))))
}
```

Finally, we box the receive side of the channel as the Response
future. Note that we need to map the channel's error to a
`hyper::Error`.

There are two principle drawbacks to this approach. First, since we
read the entire file into memory, serving many large files has the
potential to exhaust our memory. Second, we cannot start sending data
until the file has been read, increasing the latency of our response
for larger files.

### Streaming Files

We can address the problem of the single threaded approach by using a
second `mpsc::channel` to stream the file in smaller chunks. We need
to use two channels because the Response body stream cannot change the
status of the Response, but some i/o operations, e.g. `File::open`, may
return errors requiring a different response. These operations all need
to take place before the Response future is sent over the oneshot,
before the loop that streams the Response body is started.

```rust
fn stream_file(f: &str) -> Box<Future<Item = Response, Error = hyper::Error>> {
    let filename = f.to_string(); // we need to copy for lifetime issues
    let (tx, rx) = oneshot::channel();
    thread::spawn(move || {
        let mut file = match File::open(filename) {
            Ok(f) => f,
            Err(_) => {
                tx.send(Response::new()
                        .with_status(StatusCode::NotFound)
                        .with_header(ContentLength(NOTFOUND.len() as u64))
                        .with_body(NOTFOUND))
                    .expect("Send error on open");
                return;
            },
        };
```

We begin the same as the simple approach.

```rust
        let (mut tx_body, rx_body) = mpsc::channel(1);
        let res = Response::new().with_body(rx_body);
        tx.send(res).expect("Send error on successful file read");
```

Here we create an `mpsc::channel` for the response body. We create a
Response with the receive side of the `mpsc::channel` as the body, and
send it out the `oneshot::channel`.


```rust
        let mut buf = [0u8; 4096];
        loop {
            match file.read(&mut buf) {
                Ok(n) => {
                    if n == 0 {
                        // eof
                        tx_body.close().expect("panic closing");
                        break;
                    } else {
                        let chunk: Chunk = buf.to_vec().into();
                        match tx_body.send(Ok(chunk)).wait() {
                            Ok(t) => { tx_body = t; },
                            Err(_) => { break; }
                        };
                    }
                },
                Err(_) => { break; }
            }
        }
    });
```

We create a buffer, and loop through file sending buffer sized chunks
until we reach the end of the file. On errors we simply exit the
thread, but in practice some sort of logging would be appropriate.


```rust
    Box::new(rx.map_err(|e| Error::from(io::Error::new(io::ErrorKind::Other, e))))
}
```

Finally, we box up the receive channel of the oneshot as in the simple
case.

When in doubt, use the two channel streaming approach. This will work
with small data quanities at a small overhead cost, and scale safely
to large quantities of data.

## Web Services

Todo: write me!

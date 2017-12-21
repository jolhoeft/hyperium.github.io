---
title: Response Strategies
---

## Overview

The [Echo, echo, echo](./echo) guide discusses how to transform
request bodies into response bodys, and the [Handling
Posts](./handle_post) guide discusses processing more complex request
bodies bodies. We will now look into more advanced ways of generating
responses. Three use cases are considered: server files from the file
system, querying and rendering data from a database, and rendering a
response from a web service. The first two will involve processing on
a separate thread, while web services will be handled with hyper's
client support and be a purely futures based approach. A working
example of what we cover here is provided in the hyper distribution at
[responding.rs](https://github.com/hyperium/hyper/blob/master/examples/responding.rs).

We start with extracting the query information from the request (which
can be from form parameters, cookies, headers, etc, but this guide
will simply use the request path). This information is used to create
the response future (`Service::Future`, which is `Box<Future<Item =
Self::Response, Error = Self::Error>>` here). Finally, the response
body stream is assigned.

All the operations that can affect the response status code need to
take place the response future, not the response body stream. The
response body stream cannot change the status of the response. For
example, if that stream tries to open a file that does not exist, it
cannot return a 404. The file needs to be opened before the body
stream is created and the handle passed to the stream.

## File Serving

At the time of this writing, there is no cross platform way of
accessing the filesystem with futures. We will use blocking I/O in a
separate thread.

Todo: write me!

### Data Bases

Todo: write me!

## Web Services

Todo: write me!

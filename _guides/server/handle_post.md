---
title: Handling Posts
---

## Overview

A common use case is to have a web page Post to the server a chunk of
data. The data could be, for example, html form data, or Json
submitted by XmlHttpRequest. The server will parse and validate the
data, process it (possibly including service calls to a database or
web service), and render a response. Responses can include web pages,
images, or chunks of Json. A basic example of form processing is
included in the hyper distribution,
[params.rs](https://github.com/hyperium/hyper/blob/master/examples/params.rs). We
will start by discussing key aspects of that example, then address how
to modify the approach for handling other types of posted data and
rendered responses.

The basic approach to handling posts is to take the requst body (which
implements `futures::stream::Stream`) and apply a series of adapters
until it is transformed into response futures (`Service::Future`,
which is `Box<Future<Item = Self::Response, Error = Self::Error>>` in
the `params.rs` example). You might think it would be easier to
transform the request body directly into the response body. However,
that approach makes it difficult to exit early if there is a problem,
such as a malformed request or a service is unavailable. Small quickly
rendered response bodies can be generated as part of the response
future. Larger responses that may take time to stream to the client
will need to be generated in a separate stream for the response body.

## Setup

The basic structure of the `'params.rs` example is the same as in the
[Echo, echo, echo](./echo) example. Aside from handling Post, which we
discuss below, the key differences are:

Import the `url` crate for form parsing:

```rust
extern crate url;
use url::form_urlencoded;
```

Define some strings for the form we wish to process, and some error
responses:

```rust
static INDEX: &[u8] = b"<html><body><form action=\"post\" method=\"post\">Name: <input type=\"text\" name=\"name\"><br>Number: <input type=\"text\" name=\"number\"><br><input type=\"submit\"></body></html>";
static MISSING: &[u8] = b"Missing field";
static NOTNUMERIC: &[u8] = b"Number field is not numeric";
```

## Parsing the Request Body

```rust
(&Post, "/post") => {
    Box::new(req.body().concat2().
	    map(|b| {
```

First concat the request body into a future containing the entire
request body, since we cannot do anything useful until all the data is
available. Then we map the body into our response, which is the main
engine of Post handling.


```rust
    let params = form_urlencoded::parse(b.as_ref()).into_owned().collect::<HashMap<String, String>>();
```

Parse the request body. `form_urlencoded::parse` always succeeds, so
we can directly collect our form data into a HashMap. Note that this
is a simplified use case. In principle names can appear multiple times
in a form, and the values should be rolled up into a HashMap<String,
Vec<String>>. However in this example the simpler approach is
sufficient.


```rust
    let name = if let Some(n) = params.get("name") {
        n
    } else {
        return Response::new()
            .with_status(StatusCode::UnprocessableEntity)
            .with_header(ContentLength(MISSING.len() as u64))
            .with_body(MISSING);
    };
    let number = if let Some(n) = params.get("number") {
        if let Ok(v) = n.parse::<f64>() {
            v
        } else {
            return Response::new()
                .with_status(StatusCode::UnprocessableEntity)
                .with_header(ContentLength(NOTNUMERIC.len() as u64))
                .with_body(NOTNUMERIC);
        }
    } else {
        return Response::new()
            .with_status(StatusCode::UnprocessableEntity)
            .with_header(ContentLength(MISSING.len() as u64))
            .with_body(MISSING);
    };
```

Here we validate the submitted data, verifying the expected fields are
present, and that the numeric field is in fact a number. If that is
not the case we exit early with an appropriate `StatusCode`.

### Handling Json and Other Data Types

Parsing other data types follows the same pattern, although parsing
may fail. Even if you are certain your client side code will always
submit valid data, you have no garantee a Post came from your
client. To parse a Json Post:

```rust
    let json: serde_json::Value = if let Ok(j) = serde_json::from_slice(b.as_ref()) {
	    j
    } else {
        return Response::new()
            .with_status(StatusCode::BadRequest)
            .with_header(ContentLength(BADREQUEST.len() as u64))
            .with_body(BADREQUEST);
	}
```

## Generating the Response Body

```rust
    let body = format!("Hello {}, your number is {}", name, number);
    Response::new()
        .with_header(ContentLength(body.len() as u64))
        .with_body(body)
}))
```

Finally we generate the response body.

### More Complex Response Bodies

Todo:

 * spawn a thread to read the filesystem or query a database
 * web service query - can that be done purely via hyper/futures w/o a thread?

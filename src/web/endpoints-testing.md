# Endpoints testing

The Axum ecosystem offers the [axum-test](https://crates.io/crates/axum-test) crate, which allows you to perform full endpoint testing while bypassing the networking layer. This is highly convenient as it enables tests to run faster and eliminates concerns regarding port collisions.

The axum-test crate provides a [TestServer](https://docs.rs/axum-test/latest/axum_test/struct.TestServer.html) wrapper, which allows you to call endpoints "directly":

```rust,noplayground
// Create a router with the endpoints we want to test
let app = Router::new()
    .route("/give-5", get(|| async { "5" }))

// Create a test server that operates without a network listener
let server = TestServer::new(app).unwrap();

// Make a request to the endpoint "directly"
let response = server.get("/give-5").await;

// Verify the request result
response.assert_text("5");
```

---

Let's add a test for our "hello" endpoint. In doing so, we will slightly modify the router creation code to make it easier to test.

First, include axum-test in your `Cargo.toml`.

```toml
[package]
name = "test_axum"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"]}
axum = "0.8"

[dev-dependencies]
axum-test = "18"
```

The functionality from the axum-test crate is only needed for tests, so we declare it in the `[dev-dependencies]` section.

Now, let's move the router creation into a separate function so it can be called from the test, and then write the test itself:

```rust,noplayground
use std::collections::HashMap;
use axum::{Router, extract::Query, routing::get};

#[tokio::main]
async fn main() {
    let app = setup_app();
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

// Router creation
fn setup_app() -> Router {
    Router::new()
        .route("/hello", get(hello))
}

async fn hello(Query(map):Query<HashMap<String, String>>) -> String {
    if let Some(name) = map.get("name") {
        format!("Hello, {name}!")
    } else {
        String::from("Hello!")
    }
}

// This module, containing the test, will only compile when running tests.
#[cfg(test)]
mod test {
    use axum_test::TestServer;
    use axum::http::StatusCode;
    use super::*;

    #[tokio::test]
    async fn test_hello_endpoint() {
        let app = setup_app();
 
        let server = TestServer::new(app).unwrap();

        // Test call without query parameters
        let response1 = server.get("/hello").await;
        response1.assert_status(StatusCode::OK);
        response1.assert_text("Hello!");

        // Test call with a query parameter
        let response2 = server.get("/hello?name=Stas").await;
        response2.assert_status(StatusCode::OK);
        response2.assert_text("Hello, Stas!");
    }
}
```


# Extractors

An extractor is a mechanism that allows you to extract a value from (or based on) an HTTP request and inject it into a handler function.

We are already familiar with several standard extractors:

* An extractor for injecting path arguments — [Path](axum-basics.md#path-parameters)
* An extractor for injecting query parameters — [Query](axum-basics.md#query-parameters)
* An extractor for injecting the request body as a JSON document — [Json](axum-basics.md#чтение-тела-запроса)

Now, let’s look at how extractors are structured internally.

To create an extractor, you need to implement one of two traits: [FromRequestParts](https://docs.rs/axum/latest/axum/extract/trait.FromRequestParts.html) or [FromRequest](https://docs.rs/axum/latest/axum/extract/trait.FromRequest.html).

### FromRequestParts

The `FromRequestParts` trait is declared as follows:

```rust,noplayground
pub trait FromRequestParts<S>: Sized {
    type Rejection: IntoResponse;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &S,
    ) -> Result<Self, Self::Rejection>;
}
```

As we can see, it has access to the header portion of the HTTP request — [Parts](https://docs.rs/http/latest/http/request/struct.Parts.html) and the state object via its arguments. You should build your extractor on `FromRequestParts` if it only needs access to the request metadata (headers, URI, method) and does not require access to the request body.

***

To understand how this works, let's write our own simplified version of the [Query](axum-basics.md#query-parameters) extractor, which injects query parameters into a handler function.

```rust,noplayground
use std::collections::HashMap;
use axum::{Router, extract::FromRequestParts, http::request::Parts, routing::get};

struct MyQueryParams(HashMap<String, String>);

impl<S: Send + Sync> FromRequestParts<S> for MyQueryParams {
    type Rejection = ();

    async fn from_request_parts(
        parts: &mut Parts, _state: &S
    ) -> Result<Self, Self::Rejection> {
        let mut params = HashMap::new();
        // Gets the query string in the format key_1=val_1&key_2=val_2
        if let Some(query_string) = parts.uri.query() {
            // Split the query string into substrings of key=val using the & delimiter
            for pair in query_string.split("&") {
                // Extract the key and value from the key=value string
                let mut kv = pair.split("=");
                if let Some(k) = kv.next() && let Some(v) = kv.next() {
                    params.insert(k.to_string(), v.to_string());
                }
            }
        }
        Ok(MyQueryParams(params))
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/hello", get(hello));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn hello(MyQueryParams(map): MyQueryParams) -> String {
    format!("Query params: {map:?}")
}
```

If you run the server (`cargo run`) and navigate to [http://localhost:8080/hello?a=1\&b=2\&c=3](http://localhost:8080/hello?a=1\&b=2\&c=3), you should see:

```
Query params: {"b": "2", "c": "3", "a": "1"}
```

***

Now let's consider a fairly common example: an extractor that retrieves a session ID from an HTTP header.

```rust,noplayground
use axum::{ Router, extract::FromRequestParts, routing::get };
use axum::http::{StatusCode, request::Parts};

// Session ID extractor
struct SessionId(String);

impl<S: Send + Sync> FromRequestParts<S> for SessionId {
    type Rejection = StatusCode;

    async fn from_request_parts(
        parts: &mut Parts, _state: &S
    ) -> Result<Self, Self::Rejection> {
        // The session ID is passed in the 'sessionid' header
        if let Some(value) = parts.headers.get("sessionid") {
            if let Ok(string_value) = value.to_str() {
                Ok(SessionId(string_value.to_string()))
            } else {
                Err(StatusCode::BAD_REQUEST)
            }
        } else {
            Err(StatusCode::UNAUTHORIZED)
        }
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/hello", get(hello));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn hello(SessionId(session_id): SessionId) -> String {
    format!("Session ID: {session_id}")
}
```

Let's test our endpoint:

```
$ curl -i http://localhost:8080/hello
HTTP/1.1 401 Unauthorized
content-length: 0
date: Mon, 01 Dec 2025 22:20:47 GMT

$ curl -i -H "sessionid: 1111-1111-1111" http://localhost:8080/hello
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 26
date: Mon, 01 Dec 2025 22:08:03 GMT

Session ID: 1111-1111-1111
```

***

From a practical standpoint, it is much more convenient if the handler receives the actual user session object instead of just the session ID. Extractors have access not only to the HTTP request headers but also to the application state where the sessions are stored. Let's rewrite our extractor so that it first retrieves the session ID from the request headers and then fetches the session object from the state.

```rust,noplayground
use std::{collections::HashMap, sync::Arc};
use axum::{Router, extract::FromRequestParts, routing::get};
use axum::http::{StatusCode, request::Parts};
use tokio::sync::{Mutex, RwLock};

// User data stored in the session
struct SessionData {
    user_name: String,
}

struct AppState {
    // Session ID -> Session data
    sessions: RwLock<HashMap<String, Arc<Mutex<SessionData>>>>,
}

// Session extractor. Stores both the session ID and the session data object
struct Session(String, Arc<Mutex<SessionData>>);

impl FromRequestParts<Arc<AppState>> for Session {
    type Rejection = StatusCode;

    async fn from_request_parts(
        parts: &mut Parts,
        state: &Arc<AppState>,
    ) -> Result<Self, Self::Rejection> {
        if let Some(value) = parts.headers.get("sessionid") {
            if let Ok(string_value) = value.to_str() {
                let session_id = string_value.to_string();
                let read_guard = state.sessions.read().await;
                if let Some(session) = read_guard.get(&session_id) {
                    Ok(Session(session_id, session.clone()))
                } else {
                    Err(StatusCode::UNAUTHORIZED)
                }
            } else {
                Err(StatusCode::BAD_REQUEST)
            }
        } else {
            Err(StatusCode::UNAUTHORIZED)
        }
    }
}

#[tokio::main]
async fn main() {
    // Pre-populated session storage for testing our extractor
    let sessions: RwLock<HashMap<String, Arc<Mutex<SessionData>>>> = {
        let mut data = HashMap::new();
        // Test session
        data.insert(
            "1111-1111-1111".to_string(),
            Arc::new(Mutex::new(SessionData {user_name: "John Doe".to_string()})),
        );
        RwLock::new(data)
    };
    let app_state = AppState { sessions };

    let app = Router::new()
        .route("/hello", get(hello))
        .with_state(Arc::new(app_state));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn hello(Session(id, session): Session) -> String {
    format!(
        "Session ID: {id}, User name: {}",
        session.lock().await.user_name
    )
}
```

Verify that passing a non-existent session ID results in a 401 Unauthorized status:

```
$ curl -i -H "sessionid: 2222-2222-2222" http://localhost:8080/hello
HTTP/1.1 401 Unauthorized
content-length: 0
date: Mon, 01 Dec 2025 22:56:52 GMT
```

Now, let's check the successful scenario:

```
$ curl -i -H "sessionid: 1111-1111-1111" http://localhost:8080/hello
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 47
date: Mon, 01 Dec 2025 22:48:05 GMT

Session ID: 1111-1111-1111, User name: John Doe
```

### FromRequest

The `FromRequest` trait looks like this:

```rust,noplayground
pub trait FromRequest<S, M = ViaRequest>: Sized {
    type Rejection: IntoResponse;

    async fn from_request(
        req: Request<Body>,
        state: &S,
    ) -> Result<Self, Self::Rejection>
}
```

It has access to the [Request](https://docs.rs/http/latest/http/request/struct.Request.html) object, which encapsulates all request data, including the body. You should only build an extractor based on FromRequest if you specifically need access to the request body.

***

As you might have guessed, the `Json` and `Form<FormData>` types we discussed in the [section on reading request data](axum-basics.md#read-request-body) implement `FromRequest`, which allows them to be injected into handler functions.

To understand how to work with `FromRequest`, let's write an extractor capable of deserializing request data sent in both JSON and XML formats.

First, add the [serde-xml-rs](https://crates.io/crates/serde-xml-rs) dependency to your `Cargo.toml` file:

```toml
[package]
name = "test_axum"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"]}
axum = "0.8"

serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde-xml-rs = "0.8"
```

Now for the program itself — `src/main.rs`:

```rust,noplayground
use std::io::Cursor;
use axum::{Router, body::{Body, Bytes, to_bytes}, http::StatusCode, routing::post};
use axum::extract::{FromRequest, Request};
use serde::{Deserialize, de::DeserializeOwned};

// Extractor for request data
struct AnyFormat<D: DeserializeOwned>(D);

impl<S: Send + Sync, D: DeserializeOwned> FromRequest<S> for AnyFormat<D> {
    type Rejection = (StatusCode, &'static str);

    async fn from_request(
        request: Request<Body>, _state: &S
    ) -> Result<Self, Self::Rejection> {
        // Extract the parts (headers) and the body from the request
        let (parts, body) = request.into_parts();
        let Some(content_type) = parts.headers.get("content-type") else {
            return Err((StatusCode::BAD_REQUEST, "Missing content-type"));
        };
        // Read the request bytes into a buffer with a 100MB limit
        let body_bytes: Bytes = to_bytes(body, 100 * 1024 * 1024).await.unwrap();

        let result = match content_type.to_str().unwrap() {
            "application/json" => match serde_json::from_slice::<D>(&body_bytes) {
                Ok(entity) => entity,
                Err(_) => return Err((StatusCode::BAD_REQUEST, "Malformed JSON")),
            },
            "application/xml" => {
                let cursor = Cursor::new(body_bytes);
                match serde_xml_rs::from_reader::<'_, D, _>(cursor) {
                    Ok(entity) => entity,
                    Err(_) => return Err((StatusCode::BAD_REQUEST, "Malformed XML")),
                }
            }
            _ => return Err((StatusCode::BAD_REQUEST, "Unsupported format")),
        };
        Ok(AnyFormat(result))
    }
}

#[derive(Debug, Deserialize)]
struct CreateUserRequest {
    name: String,
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/users", post(create_user));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn create_user(AnyFormat(req): AnyFormat<CreateUserRequest>) -> String {
    format!("Received: {req:?}")
}
```

Now, after starting the server, we can send a request to the same endpoint http://localhost:8080/users using JSON:

```
$ curl -i -X POST -H "content-type: application/json" -d '{"name":"John Doe"}' \
    http://localhost:8080/users
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 48
date: Tue, 02 Dec 2025 00:31:16 GMT

Received: CreateUserRequest { name: "John Doe" }
```

And also in XML format:

```
$ curl -i -X POST -H "content-type: application/xml" \
    -d '<CreateUserRequest><name>John Doe</name></CreateUserRequest>' \
    http://localhost:8080/users
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 48
date: Tue, 02 Dec 2025 00:56:08 GMT

Received: CreateUserRequest { name: "John Doe" }
```


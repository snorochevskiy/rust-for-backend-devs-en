# Middleware

**Middleware** is functionality that is called before a request handler function and also has the ability to process the result of that handler after it finishes. You could say that middleware "wraps" the request handler call.

<pre class="ascii-diagram">
╭────╮     ┌──────┐ request ┌──────────┐ request ┌──────────┐
│    ├────▶│      ├────────▶│          ├────────▶│          │
│Сеть│ TCP │ Axum │         │middleware│         │обработчик│
│    │◀────┤роутер│◀────────┤          │◀────────┤ запроса  │
╰────╯     └──────┘ response└──────────┘ response└──────────┘
</pre>

If we want to perform specific actions before processing any request, middleware is the mechanism that allows us to do just that.

Furthermore, we can create multiple middlewares that will execute in a chain, running both before and after the request handler.

<pre class="ascii-diagram">
    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
───▶│          ├───▶│          ├───▶│          ├───▶│          ├───▶
    │middleware│    │middleware│    │middleware│    │middleware│
◀───┤    1     │◀───┤    2     │◀───┤    3     │◀───┤    4     │◀───
    └──────────┘    └──────────┘    └──────────┘    └──────────┘
</pre>

As you can see from the diagram above, middleware objects are stacked like layers wrapping the handler. This is why registering middleware in a router is referred to as adding a **layer**.

## Simple Middleware

The simplest form of middleware is a function that accepts a [Request](https://docs.rs/axum/latest/axum/extract/type.Request.html) object (the HTTP request) and a [Next](https://docs.rs/axum/latest/axum/middleware/struct.Next.html) object (a reference to the next middleware in the chain or the final handler). It returns a `Response` object obtained by calling the next item in the chain.

```rust,noplayground
async fn my_middleware(request: Request, next: Next) -> Response {
    // ... actions before processing the request
    
    // Call the next handler in the chain
    let response = next.run(request).await;
    
    // ... actions after processing the request

    // Return the result from the middleware
    response
}
```

To use this function, you need to:

* Convert it into middleware using the [from_fn](https://docs.rs/axum/latest/axum/middleware/fn.from_fn.html) function
* Register it in the router using the [layer](https://docs.rs/axum/latest/axum/struct.Router.html#method.layer) method

Once registered, the middleware will execute for all endpoints defined in that router.

```rust,noplayground
let app = Router::new()
    .route("/path", get(handler_function))
    .layer(middleware::from_fn(my_middleware));
```

To see how this works in practice, let's write a classic example: middleware that logs the execution time of a handler.

```rust,noplayground
use axum::{Router, extract::Request, response::Response, routing::get};
use axum::middleware::{self, Next};
use tokio::time::Instant;

async fn log_exec_time(request: Request, next: Next) -> Response {
    let start = Instant::now();
    let response = next.run(request).await;
    tracing::info!("Request took: {} micros", start.elapsed().as_micros());
    response
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        .init();

    let app = Router::new()
        .route("/hello", get(hello))
        .layer(middleware::from_fn(log_exec_time));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    "Hello!"
}
```

Start the server:

```
$ cargo run
```

Then visit [http://localhost:8080/hello](http://localhost:8080/hello) in your browser.

You should see an INFO level log entry in your console from the application:

```
2025-12-02T23:09:30.342625Z  INFO test_axum: Request took: 52 micros
```

## Layers

Now, let's explore how different middlewares interact with one another.

We will add two middlewares to our "hello" server, each designed to:

* Print a log message before calling the next handler in the chain.
* Print another log message after the next handler in the chain has finished its work.

```rust,noplayground
use axum::middleware::{Next, from_fn};
use axum::{Router, extract::Request, routing::get};

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        .init();

    let app = Router::new()
        .route("/hello", get(hello))
        .layer(from_fn(async |request: Request, next: Next| {
            tracing::info!("Middleware-1: before call");
            let response = next.run(request).await;
            tracing::info!("Middleware-1: after call");
            response
        }))
        .layer(from_fn(async |request: Request, next: Next| {
            tracing::info!("Middleware-2: before call");
            let response = next.run(request).await;
            tracing::info!("Middleware-2: after call");
            response
        }));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    tracing::info!("Request handler");
    "Hello!"
}
```

Run the server and navigate to [http://localhost:8080/hello](http://localhost:8080/hello)

```
stas@slim:~/dev/proj/rust/test_axum$ cargo run
   Compiling test_axum v0.1.0
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 1.16s
     Running `target/debug/test_axum`

2025-12-03T22:36:42.563720Z  INFO test_axum: Middleware-2: before call
2025-12-03T22:36:42.563776Z  INFO test_axum: Middleware-1: before call
2025-12-03T22:36:42.563800Z  INFO test_axum: Request handler
2025-12-03T22:36:42.563823Z  INFO test_axum: Middleware-1: after call
2025-12-03T22:36:42.563834Z  INFO test_axum: Middleware-2: after call
```

As you can see, middlewares wrap around each other like nesting dolls (matryoshkas). Consequently, the middleware that was added to the router last becomes the first one in the execution chain.

## Working with State

Often, middleware logic may need access to data stored in the application state. In this case, the middleware function will take an additional argument: `State`.

```rust,noplayground
async fn my_middleware(state: State<MyState>, request: Request, next: Next) -> Response {
    // Actions before processing the request
    let response = next.run(request).await;
    // Actions after processing the request
    response
}
```

Such a function is converted into middleware using the [from_fn_with_state](https://docs.rs/axum/latest/axum/middleware/fn.from_fn_with_state.html) function.

```rust,noplayground
let app = Router::new()
    .route("/path", get(handler_function))
    .with_state(state.clone())
    .layer(middleware::from_fn_with_state(state, my_middleware));
```

As you may have noticed, the state must be passed separately to both `with_state` and `from_fn_with_state`. While this might seem slightly redundant, it allows you to use different state objects for the middleware and the router if necessary.

As an example, let's write a middleware that extracts a session ID from an HTTP header, retrieves the corresponding user session object from the state, and places it into a [task-local variable](../async/tokio.md#task-local). This task-local variable is then used within the request handler function.

```rust,noplayground
use std::{collections::HashMap, sync::Arc};
use axum::{Router, extract::{Request, State}, http::StatusCode, routing::get};
use axum::middleware::{self, Next};
use axum::response::{IntoResponse, Response};
use tokio::sync::{Mutex, RwLock};

tokio::task_local! {
    pub static SESSION: Arc<Mutex<SessionData>>;
}

// User session data
struct SessionData {
    user_name: String,
}

// Backend state storing sessions
struct AppState {
    sessions: RwLock<HashMap<String, Arc<Mutex<SessionData>>>>,
}

async fn set_session_for_request(
    State(state): State<Arc<AppState>>, request: Request, next: Next
) -> Response {
    // Retrieve the session object
    let session = if let Some(value) = request.headers().get("sessionid") {
        if let Ok(string_value) = value.to_str() {
            let session_id = string_value.to_string();
            let read_guard = state.sessions.read().await;
            if let Some(session) = read_guard.get(&session_id) {
                session.clone()
            } else {
                return StatusCode::UNAUTHORIZED.into_response();
            }
        } else {
            return StatusCode::BAD_REQUEST.into_response();
        }
    } else {
        return StatusCode::UNAUTHORIZED.into_response();
    };

    // Store the session object in a task-local variable
    let response = SESSION.scope(session, async {
            next.run(request).await // call the next handler in the chain
        }).await;
    response
}

#[tokio::main]
async fn main() {
    let sessions: RwLock<HashMap<String, Arc<Mutex<SessionData>>>> = {
        let mut data = HashMap::new();
        data.insert( // Create a test user session
            "1111-1111-1111".to_string(),
            Arc::new(Mutex::new(SessionData {
                user_name: "John Doe".to_string(),
            })),
        );
        RwLock::new(data)
    };
    let app_state = Arc::new(AppState { sessions });

    let app = Router::new()
        .route("/hello", get(hello))
        .with_state(app_state.clone())
        .layer(middleware::from_fn_with_state(app_state, set_session_for_request));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> String {
    // Extract the session from the task-local variable
    let session = SESSION.with(|session| session.clone());
    // Use the session
    format!("Hello, {}!", session.lock().await.user_name)
}
```

Let's test our endpoint to ensure the middleware correctly places the session into the task-local variable accessible from the handler.

```
$ curl -i -H "sessionid: 1111-1111-1111" http://localhost:8080/hello

HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 15
date: Wed, 03 Dec 2025 14:03:22 GMT

Hello, John Doe!
```

## Standard Middleware

The Axum ecosystem provides several standard middlewares for tasks like CORS, compression, timeouts, and more. However, before we look at how to use them, we need to get acquainted with the [Tower](tower.md) library.


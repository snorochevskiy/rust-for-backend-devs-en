# Backend State

In the previous chapter, we learned how to create endpoints that only work with values received from the request. However, in real-world backend applications, there is usually some form of "application state". This state might contain configuration settings, a database connection pool, HTTP or gRPC clients for communicating with other services, metrics data, caches, and more.

To handle this backend state, Axum provides the [State](https://docs.rs/axum/latest/axum/extract/struct.State.html). The workflow is as follows:

1\) We create an object of an arbitrary structure and add it to the router configuration using the [with_state](https://docs.rs/axum/latest/axum/struct.Router.html#method.with_state) method.

```rust,noplayground
struct AppState {
  // fields
}

#[tokio::main]
async fn main() {
    let my_app_data = Arc::new(AppState { ... });

    let app = Router::new()
        .route("/something", get(handler))
        .with_state(my_app_data);
    ...
}
```

2\) Then, using the `State` wrapper, this object can be injected into any handler function registered within that router.

```rust,noplayground
async fn handler(State(my_app_data): State<Arc<AppState>>) -> Response { ... } 
```

For example, let's create a simple backend with a counter endpoint. Every time we make a GET request to http://localhost:8080/count, the counter value will increment, and we will receive a string containing the new counter value in response.

```rust,noplayground
use std::sync::{ Arc, atomic::{AtomicU64, Ordering} };
use axum::{Router, extract::State, routing::get};

// This struct defines our backend state
struct AppState {
    counter: AtomicU64,
}

#[tokio::main]
async fn main() {
    let shared_state = Arc::new(AppState {
        counter: AtomicU64::new(0),
    });

    let app = Router::new()
        .route("/count", get(tick_counter))
        .with_state(shared_state);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn tick_counter(State(state): State<Arc<AppState>>) -> String {
    let prev_value = state.counter.fetch_add(1, Ordering::Relaxed);
    format!("New value: {}", prev_value + 1)
}
```

The first visit to [http://localhost:8080/count](http://localhost:8080/count) should display "New value: 1", the second "New value: 2", and so on.

***

You might have a logical question: why not just use a global counter variable? Using `State` provides several advantages:

* Clarity: It is immediately obvious which data belongs to the application state.
* Testability: It makes testing endpoints significantly easier.
* Flexibility: In Axum, an application can have multiple routers, and each can be assigned its own state object—something we will explore in the [next chapter](routing.md#merging-routers)

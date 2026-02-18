# Routing

In this chapter, we will take a closer look at endpoint routing capabilities.

### Fallback

If the path from the URL entered by the user does not match any endpoint registered in the router, Axum will simply respond with an HTTP 404 code by default.

If you need to handle such requests to non-existent endpoints "manually," you can define a fallback handler. A fallback is a regular request handler function that is registered in the router using the [fallback](https://docs.rs/axum/latest/axum/struct.Router.html#method.fallback) method and is called for all requests for which no matching endpoint was found.

As an example, let's create a fallback handler that simply displays the HTTP method and the request URL.

```rust,noplayground
use axum::{Router, http::request::Parts, routing::get};

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/hello", get(hello))
        .fallback(my_fallback);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    "Hello!"
}

async fn my_fallback(parts: Parts) -> String {
    format!("Method: {}\nURL: {}", parts.method, parts.uri)
}
```

After running the program, if you navigate to [http://localhost:8080/hello](http://localhost:8080/hello), you will see "Hello!". However, attempting to visit any other path, such as [http://localhost:8080/non-existing-page?a=1\&b=2](http://localhost:8080/non-existing-page?a=1\&b=2), will result in a response like this:

```
Method: GET
URL: /non-existing-page?a=1&b=2
```

### Merging Routers { #merging-routers }

An endpoint router can be built by composing multiple sub-routers. This means we can define one set of endpoints in one router object, another set in a different one, and then "glue" them together into a main router using the [merge](https://docs.rs/axum/latest/axum/struct.Router.html#method.merge) method.

For example, let's create one router for user-related endpoints and another for product-related endpoints.

```rust,noplayground
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let users_router = Router::new()
        .route("/users", get(list_users));

    let products_router = Router::new()
        .route("/products", get(list_products));

    let app = Router::new()
        .merge(users_router)
        .merge(products_router);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn list_users() -> &'static str {
    "Users"
}

async fn list_products() -> &'static str {
    "Products"
}
```

Splitting endpoints into separate routers makes your code significantly more modular. However, readability isn't the only advantage here. This approach also allows you to define separate `State` objects for each router, providing isolation for different parts of your application's state.

Let’s modify our example so that each sub-router uses its own state object.

```rust,noplayground
use std::sync::Arc;

use axum::{Router, extract::State, routing::get};

#[derive(Debug)]
struct UserState {}

#[derive(Debug)]
struct ProductState {}

#[tokio::main]
async fn main() {
    let users_router = Router::new()
        .route("/users", get(list_users))
        .with_state(Arc::new(UserState {}));

    let products_router = Router::new()
        .route("/products", get(list_products))
        .with_state(Arc::new(ProductState {}));

    let app = Router::new().merge(users_router).merge(products_router);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn list_users(State(state): State<Arc<UserState>>) -> String {
    format!("Users endpoint. State: {state:?}")
}

async fn list_products(State(state): State<Arc<ProductState>>) -> String {
    format!("Products endpoint. State: {state:?}")
}
```

If we run the application and visit [http://localhost:8080/users](http://localhost:8080/users), we will see: `Users endpoint. State: UserState`.

If we visit [http://localhost:8080/products](http://localhost:8080/products), it should display: `Products endpoint. State: ProductState`.

As you can see, endpoints interact with the state object that was specifically assigned to their sub-router. But what happens if we add another state object at the top-level merging router?

```rust,noplayground
#[derive(Debug)]
struct MainState {}

let app = Router::new()
    .merge(users_router)
    .merge(products_router)
    .with_state(Arc::new(MainState));
```

Actually, nothing changes: handlers use the state object declared "closest" to them in the hierarchy.

However, if you were to remove the state declared at the `users_router` level and instead provide a `UserState` object at the main router level, the [http://localhost:8080/users](http://localhost:8080/users) endpoint would then use the state from the main router.

### Nesting Sub-routers

Nesting is similar to the merging concept discussed previously, with one key difference: when nesting, the endpoint paths from the sub-router are prefixed with a specific path.

For example, consider the following router:

```rust,noplayground
let users_router = Router::new().route("/users", get(list_users));
```

If we nest it into the main router like this:

```rust,noplayground
let main_router = Router::new().nest("/api", users_router);
```

then the `list_users` handler function will be triggered by a request to the URL http://localhost:8080/api/users.

Why is this useful? It further enhances code modularity. For instance, you can move specific endpoints into a separate library and nest them into a router in a different application without worrying about path conflicts with other existing endpoints.

Let's look at an example. We will create two routers containing endpoints with conflicting paths. By using different prefixes, we can nest both routers into the main application router.

```rust,noplayground
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let users_v1_router = Router::new().route("/users", get(list_users_v1));

    let users_v2_router = Router::new().route("/users", get(list_users_v2));

    let app = Router::new()
        .nest("/api/v1", users_v1_router)
        .nest("/api/v2", users_v2_router);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn list_users_v1() -> &'static str {
    "Users endpoint Version 1"
}

async fn list_users_v2() -> &'static str {
    "Users endpoint Version 2"
}
```

Now, navigating to [http://localhost:8080/api/v1/users](http://localhost:8080/api/v1/users) will display "Users endpoint Version 1", while navigating to [http://localhost:8080/api/v2/users](http://localhost:8080/api/v2/users) will display "Users endpoint Version 2".


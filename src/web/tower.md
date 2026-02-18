# Tower

It’s time to talk about the [Tower](https://crates.io/crates/tower). library. This library provides a toolkit for building clients and servers from separate, reusable building blocks.

The first thing we need to do is add the [tower](https://crates.io/crates/tower) and [tower_http](https://crates.io/crates/tower-http) dependencies to our `Cargo.toml`.

```toml
[package]
name = "test_axum"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = { version = "1", features = ["full"]}
axum = "0.8"
tower = { version = "0.5", features = ["full"] }
tower-http = { version = "0.6", features = ["cors", "fs"] }

tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

The core type provided by Tower is the [Service](https://docs.rs/tower/latest/tower/trait.Service.html) trait, which serves as an abstraction over an asynchronous function.

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    // Works on the same principle as 'poll' in the Future trait:
    // * Returns Poll::Ready(Ok(())) if the service is ready to process requests
    //   and 'call' can be invoked.
    // * Returns Poll::Pending if it is not ready. The executor will be notified
    //    of readiness via the Context.
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;

    // Used to directly invoke the service's functionality.
    fn call(&mut self, req: Request) -> Self::Future;
}
```

A `Service` is an abstraction over an async function that takes an object of some generic `Request` type as an argument and returns either a generic `Response` or an error.

You can think of a service as: `async (Request) => Result<Response, Error>`.

This abstraction is exactly what is needed to build a network server. For example:

* Request handler functions that we register in the router are essentially async functions that take an HTTP Request and return an HTTP Response.
* Middleware, which we covered in the previous chapter, are also essentially async functions that take an HTTP Request and return an HTTP Response.

So, we've established that entities like request handlers and middleware can be abstracted through the `Service` trait. Но what does this actually give us?

The key is that if different handlers are abstracted into a <ins>single</ins> interface, they become universal building blocks. From these, you can construct request-processing chains. This means you can create universal implementations for things like retries, back-pressure, request filtering, CORS, compression, timeouts, and more. This is exactly what the Tower library is: the `Service` trait and a vast collection of ready-made, universal building blocks built upon it.

This is particularly interesting for us because Axum allows you to use `Service` implementations as both request handlers and middleware.

## Request Handler as a Service

Up to this point, we have created endpoints by registering handler functions in the router using the [route](https://docs.rs/axum/latest/axum/struct.Router.html#method.route) method. However, the router provides another method — [route_service](https://docs.rs/axum/latest/axum/struct.Router.html#method.route_service) which registers a `Service` as a handler:

```rust
pub fn route_service<T>(self, path: &str, service: T) -> Self
where
    T: Service<Request, Error = Infallible> + Clone + Send + Sync + 'static,
    T::Response: IntoResponse,
    T::Future: Send + 'static,
```

The `where` block states that for a type to act as a request handler, it must have a `Service` implementation where:

* The input argument type is `axum::extract::Request`.
* The `call` method returns an object of a type that implements `IntoResponse`, wrapped only in `Ok`, not `Err`. If an error occurs, you can return an `Ok(Response)` object with the appropriate HTTP status code, but it must be an `Ok`.

In other words:

```rust,noplayground
impl Service<Request> for МойСервис {
  type Response = Response;
  type Error = Infallible;
  type Future = Pin<Box<dyn Future<Output=Result<Self::Response,Self::Error>>+Send>>;

  fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
    ...
  }

  fn call(&mut self, req: Request) -> Self::Future {
    ...
  }
}
```

> [!TIP]
> [std::convert::Infallible](https://doc.rust-lang.org/std/convert/enum.Infallible.html) is an error type defined as an empty enum, making it impossible to create an instance of this type. It is used in situations where a method defined in a trait returns a `Result`, but specific implementations of that trait are required to never return an error.

For our example, let's take our traditional "Hello" server, but this time the request handler will be an object whose type implements the `Service` trait instead of a function.

```rust,noplayground
use axum::{Router, extract::Request, response::{Response,IntoResponse}};
use axum::http::{Method, StatusCode};
use std::{future::Future, pin::Pin, task::{Context, Poll}, convert::Infallible};
use tower::Service;

#[derive(Clone)]
struct HelloService {
    greeting: String,
}

impl Service<Request> for HelloService {
    type Response = Response;
    type Error = Infallible;
    type Future = Pin<Box<dyn Future<Output=Result<Self::Response,Self::Error>>+Send>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        Poll::Ready(Ok(()))
    }

    fn call(&mut self, req: Request) -> Self::Future {
        if req.method() == Method::GET {
            let greeting = self.greeting.clone();
            Box::pin(async move { Ok(greeting.into_response()) })
        } else {
            Box::pin(async move { Ok(StatusCode::METHOD_NOT_ALLOWED.into_response()) })
        }
    }
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route_service("/hello", HelloService { greeting: "Hello!".to_string() });
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

Run the server and navigate to [http://localhost:8080/hello](http://localhost:8080/hello). You should see the same "Hello!" message.

It is easy to see that creating a request handler from a type that implements `Service` is more labor-intensive than creating one from a function. However, this approach provides more flexibility, as it allows our handler object to maintain internal state.

## Middleware as a Service

As we know from the previous chapter, middleware is registered in the router using the [layer](https://docs.rs/axum/latest/axum/struct.Router.html#method.layer) method, which we are finally ready to examine in more detail. It has the following signature:

```rust,noplayground
pub fn layer<L>(self, layer: L) -> Router<S>
where
    L: Layer<Route> + Clone + Send + Sync + 'static,
    L::Service: Service<Request> + Clone + Send + Sync + 'static,
    <L::Service as Service<Request>>::Response: IntoResponse + 'static,
    <L::Service as Service<Request>>::Error: Into<Infallible> + 'static,
    <L::Service as Service<Request>>::Future: Send + 'static
```

Here, [Layer](https://docs.rs/tower-layer/latest/tower_layer/trait.Layer.html) is a trait for types that return an object implementing the `Service` trait. You could say that `Layer` is a factory for a `Service`. The Layer trait itself is declared as follows:

```rust,noplayground
pub trait Layer<S> {
    type Service;

    fn layer(&self, inner: S) -> Self::Service;
}
```

The `layer` method takes an `inner` argument, which represents either the next middleware in the chain or the final request handler. The method returns a middleware object that will be embedded into the processing chain before the link passed in the inner argument.

As an example, let’s rewrite our middleware from the previous chapter — the one that logs the execution time of a request handler — as a service.

```rust,noplayground
use std::{pin::Pin, task::{Context, Poll}};
use axum::{Router, extract::Request, response::Response, routing::get};
use tokio::time::Instant;
use tower::{Layer, Service};

// Middleware that measures request execution time.
#[derive(Clone)]
struct ExecTimeLogService<S> {
    // The next middleware/request handler
    next_handler: S,
}

impl<S> Service<Request> for ExecTimeLogService<S>
where
    S: Service<Request, Response = Response> + Send + 'static,
    S::Future: Send + 'static,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = Pin<Box<dyn Future<Output=Result<Self::Response,Self::Error>>+Send>>;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.next_handler.poll_ready(cx)
    }

    fn call(&mut self, request: Request) -> Self::Future {
        let start = Instant::now();
        let future = self.next_handler.call(request);
        Box::pin(async move {
            let response = future.await?;
            tracing::info!("Request took: {} micros", start.elapsed().as_micros());
            Ok(response)
        })
    }
}

// Layer implementation responsible for injecting our middleware 
// into the request processing chain
#[derive(Clone)]
struct ExecTimeLogLayer;

impl<S> Layer<S> for ExecTimeLogLayer {
    type Service = ExecTimeLogService<S>;

    // Axum calls this method to get a middleware object for further 
    // integration into the request processing chain.
    // 'inner' is the next middleware or handler in the chain.
    fn layer(&self, inner: S) -> Self::Service {
        ExecTimeLogService { next_handler: inner }
    }
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        .init();
    let app = Router::new()
        .route("/hello", get(hello))
        .layer(ExecTimeLogLayer);
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    "Hello!"
}
```

Let's run our application and navigate to [http://localhost:8080/hello](http://localhost:8080/hello):

```
$ cargo run

   Compiling test_axum v0.1.0
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 2.00s
     Running `target/debug/test_axum`
2025-12-05T17:32:48.512545Z  INFO test_axum: Request took: 82 micros
```

Just like with request handlers, creating middleware by implementing the `Service` trait offers more flexibility, as it allows the middleware object to maintain internal state.

## Standard Services

In the previous chapter, we mentioned that Axum supports a variety of standard middleware (and services) based on the Tower infrastructure. Now we are ready to explore them.

### The ServeFile Service

The [ServeFile](https://docs.rs/tower-http/latest/tower_http/services/struct.ServeFile.html) service allows you to serve a file as a response to a request.

For example, let's create an endpoint that returns an HTML file. First, create an `index.html` file in the root of your project:

```html
<html>
    <body>
        <h1>Hello</h1>
    </body>
</html>
```

Now, update `src/main.rs`:

```rust,noplayground
use axum::Router;
use tower_http::services::ServeFile;

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route_service("/index", ServeFile::new("index.html"));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

After starting the server and navigating to [http://localhost:8080/index](http://localhost:8080/index), you should see the page from `index.html`.


### The CorsLayer Middleware

If you are writing an API server intended to be called from a browser, you will inevitably encounter the CORS (Cross-Origin Resource Sharing) issue. If the domain where the website is hosted differs from the domain of the API server, the browser will block the calls for security reasons.

To allow a page loaded from Domain X to perform requests to an API on Domain Y, the API server (Y) must explicitly permit calls from Domain X. While we could write our own middleware to set the necessary `Access-Control-Allow-Origin` headers, there is no need to reinvent the wheel—there is a ready-made middleware for this called [CorsLayer](https://docs.rs/tower-http/latest/tower_http/cors/struct.CorsLayer.html).

In the example below, we allow cross-origin calls to our endpoints if the calls originate from pages loaded from `http://mydomain.com` or `http://api.mydomain.com`.

```rust,noplayground
use axum::{Router, routing::get};
use tower_http::cors::CorsLayer;

#[tokio::main]
async fn main() {
    let allowed_origins = [
        "http://mydomain.com".parse().unwrap(),
        "http://api.mydomain.com".parse().unwrap(),
    ];
    let app = Router::new()
        .route_service("/api/hello", get(hello))
        .layer(CorsLayer::new().allow_origin(allowed_origins));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    "Hello!"
}
```

## Other Services

By now, you should have a good grasp of how to use services and middleware based on Tower. To wrap things up, here are some of the most commonly used ones:

* [ServeDir](https://docs.rs/tower-http/latest/tower_http/services/struct.ServeDir.html) service – Similar to `ServeFile`, but serves requested files from a specified directory instead of a single file.
* [Redirect](https://docs.rs/tower-http/latest/tower_http/services/struct.Redirect.html) service – Redirects a request to another URL.
* [CompressionLayer](https://docs.rs/tower-http/latest/tower_http/compression/struct.CompressionLayer.html) middleware – Compresses the response using a specified codec (gzip, deflate, zstd, etc.).
* [RequestDecompressionLayer](https://docs.rs/tower-http/latest/tower_http/decompression/struct.RequestDecompressionLayer.html) middleware – Automatically decodes a compressed request body.
* [NormalizePathLayer](https://docs.rs/tower-http/latest/tower_http/normalize_path/struct.NormalizePathLayer.html) middleware – Removes unnecessary trailing slashes / from the URL path.
* [RequestBodyLimitLayer](https://docs.rs/tower-http/latest/tower_http/limit/struct.RequestBodyLimitLayer.html) middleware – Rejects requests with a 413 status code if the body exceeds a specified size.
* [TimeoutLayer](https://docs.rs/tower-http/latest/tower_http/timeout/struct.TimeoutLayer.html) middleware – Sets a maximum execution time for an endpoint, returning a specific HTTP code if it expires.
* [TraceLayer](https://docs.rs/tower-http/latest/tower_http/trace/struct.TraceLayer.html) middleware – Logs the moment a request is received and its total execution time.
* [InFlightRequestsLayer](https://docs.rs/tower-http/latest/tower_http/metrics/struct.InFlightRequestsLayer.html) middleware – Tracks the "number of requests currently being processed" metric.


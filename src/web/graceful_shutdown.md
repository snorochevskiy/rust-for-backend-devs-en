# Server Shutdown

We have covered how to create an HTTP server; now let's explore how to shut it down properly.

## graceful shutdown

By default, if a program receives a **signal** to terminate, it stops immediately. If an Axum server is running, any requests being processed at that moment are abruptly interrupted.

> [!NOTE]
> In this context, we are referring to signals sent by the operating system in response to an action intended to close the program. For example, in Unix-like systems, the following signals are sent:
> 
> * `SIGTERM` if the parent process has terminated.
> * `SIGINT` if the user presses Ctrl+C in the terminal where the program is running.
> 
> Both of these signals lead to an immediate shutdown of the program by default.\
> (Signals can also be sent programmatically or using the [kill](https://man7.org/linux/man-pages/man1/kill.1.html) utility)

If a server performs tasks that should not be interrupted before completion, we use what is known as a graceful shutdown. This approach ensures the server doesn't quit instantly but instead waits for all currently active requests to finish.

To implement a graceful shutdown, we use the [with_graceful_shutdown](https://docs.rs/axum/latest/axum/serve/struct.Serve.html#method.with_graceful_shutdown) method as follows:

```rust,noplayground
axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_future)
    .await
    .unwrap();
```

How does it work? Adding the `.with_graceful_shutdown()` call changes the server's behavior: as soon as the server starts, it begins waiting for the `shutdown_future` to complete. The "shutdown future" can be any object whose type implements the `Future` trait we are already familiar with. Once this future completes, Axum begins the shutdown process:

1. It immediately stops accepting new network requests.
2. It waits for all currently executing requests to finish.
3. It shuts down the server entirely.

First, let's look at how this shutdown future works in practice with the following example:

```rust,noplayground
use axum::{Router, routing::get};
use std::time::Duration;

#[tokio::main]
async fn main() {
    let app = Router::new().route("/hello", get(async || "Hello!"));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(tokio::time::sleep(Duration::from_secs(5)))
        .await
        .unwrap();
}
```

If you run this application (`cargo run`), it will run for exactly 5 seconds and then exit. Why? As mentioned, calling `.with_graceful_shutdown(shutdown_future)` tells Axum to wait for that future to finish before initiating the shutdown. In the example above, we used the result of the [sleep](https://docs.rs/tokio/latest/tokio/time/fn.sleep.html) function from the Tokio library as our future. This function returns a future that completes after the specified time interval—in our case, 5 seconds.

This example demonstrates that any future can serve as a shutdown signal, giving us the flexibility to implement various termination mechanisms.

> [!IMPORTANT]
> When we set a shutdown future using `with_graceful_shutdown`, we are not overriding the OS's ability to kill the program immediately via `SIGTERM` or `SIGINT`. Rather, we are defining an additional "off-switch" that we can control programmatically.

## Signals

As previously mentioned, by default, when an Axum application receives a `SIGTERM` or `SIGINT` signal, a default handler is triggered that immediately terminates the entire application. In this case, a graceful shutdown does not occur, which is undesirable for our purposes.

If we want graceful shutdown to work when receiving `SIGTERM` and `SIGINT`, we need to create our own shutdown future that completes upon receiving these signals, thereby initiating a correct shutdown sequence.

***

To intercept `SIGINT`, the Tokio library provides the [ctrl_c](https://docs.rs/tokio/latest/tokio/signal/fn.ctrl_c.html) function. This function overrides the default `SIGINT` handler, preventing the default behavior of immediate application termination.

> [!TIP]
> `SIGINT` is a signal specific to UNIX-like operating systems; however, Windows has an analog—the `CTRL_C_EVENT`, which follows the same rules. When running on Windows, the `ctrl_c` function will intercept `CTRL_C_EVENT`.

You can try running this program:

```rust,noplayground
#[tokio::main]
async fn main() {
    tokio::signal::ctrl_c().await.unwrap();
    println!("After Ctrl+C had been pressed");
}
```

After pressing Ctrl+C, you will see the text "After Ctrl+C had been pressed" in the console. However, if you run this code:

```rust,noplayground
#[tokio::main]
async fn main() {
    tokio::time::sleep(std::time::Duration::from_hours(1)).await;
    println!("After Ctrl+C had been pressed");
}
```

and press Ctrl+C, you won't see anything. This proves that setting up a handler for SIGINT by calling the `ctrl_c` function overrides the default `SIGINT` handler.

***

The second signal we need to intercept is `SIGTERM`. This signal is typically sent to a program either when the operating system is shutting down or when the program's parent process is terminating.

> [!TIP]
> From a backend development perspective, it's important to note that when Kubernetes begins to terminate a pod, it sends a `SIGTERM` to the program running in the container. By default, Kubernetes gives the program 30 seconds to shut down gracefully, after which it sends a `SIGKILL`, which terminates the program immediately at the OS kernel level.

Unlike the `SIGINT` signal, there is no equivalent for `SIGTERM` on Windows. Therefore, code to intercept both signals usually looks like this:

```rust,noplayground
use tokio::signal;

#[tokio::main]
async fn main() {
    let ctrl_c = signal::ctrl_c();

    // SIGTERM is only available on UNIX-like systems
    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("Cannot create SIGTERM handler")
            .recv()
            .await;
    };
    
    // On non-UNIX systems, we replace the SIGTERM wait with an infinite wait
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();
    
    // Wait for either Ctrl+C or SIGTERM
    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
    println!("After Ctrl+C or SIGTERM");
}
```

Here we use the `select` macro, which was covered in the chapter on [Tokio](../async/tokio.md#channels).

***

Now let's implement a graceful server shutdown when Ctrl+C is pressed in the console (leaving out `SIGTERM` for now to keep the code concise):

```rust,noplayground
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/hello", get(async || "Hello!"));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

async fn shutdown_signal() {
    tokio::signal::ctrl_c().await.unwrap()
}
```

This version of the server shuts down correctly upon pressing Ctrl+C. Before fully stopping, Axum will wait for all ongoing HTTP requests to finish processing.

You can easily verify that the server waits for all requests to complete. Let’s modify our "hello" endpoint so that it never completes:

```rust,noplayground
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let app = Router::new().route(
        "/hello",
        get(async || {
            std::future::pending::<()>().await; // Will never complete
            "Hello!"
        }),
    );
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

async fn shutdown_signal() {
    tokio::signal::ctrl_c().await.unwrap()
}
```

Now, if we start the server and navigate to [ http://localhost:8080/hello](http://localhost:8080/hello), we initiate a request that will never finish. If we then press Ctrl+C in the console where the server is running, we initiate a graceful shutdown that will simply hang, waiting for the "hello" request to finish.

The solution to this hanging problem is obvious: we need to add a timeout to the requests. From the previous chapter, we know there is a standard Tower middleware that implements timeouts — [TimeoutLayer](https://docs.rs/tower-http/latest/tower_http/timeout/struct.TimeoutLayer.html).

Here is the final example including a timeout and handling for both `SIGINT` and `SIGTERM`:

```rust,noplayground
use axum::Router;
use tokio::signal;

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route(
            "/hello",
            get(async || {
                std::future::pending::<()>().await; // Will never complete
                "Hello!"
            }),
        )
        .layer(TimeoutLayer::with_status_code(
            StatusCode::REQUEST_TIMEOUT,
            Duration::from_secs(10),
        ));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

async fn shutdown_signal() {
    let ctrl_c = signal::ctrl_c();

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("Cannot create SIGTERM handler")
            .recv()
            .await;
    };
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

## Stopping Background Tasks

Finally, let’s explore a scenario where, in addition to the HTTP server, we have a background task (a Tokio task) that receives messages through a channel sent from an HTTP request handler. To achieve a graceful shutdown, we need to follow a specific sequence: first, stop accepting new HTTP requests; then, process any remaining messages left in the channel; and only after that, terminate the background task.

For this example, we’ll write a simple server with a single endpoint. This endpoint takes a word from the URL path and sends it into a channel. The background task will simply read messages from the channel and print them to the console.

```rust,noplayground
use axum::{Router, http::StatusCode, routing::get};
use axum::extract::{Path, State};
use std::time::Duration;
use tokio::{signal, sync::{broadcast, mpsc}};

#[tokio::main]
async fn main() {
    // Channel to notify background tasks that they need to shut down
    let (shutdown_snd, mut shutdown_rcv) = broadcast::channel::<()>(1);
    // Channel through which the endpoint sends messages to the background task
    let (word_snd, mut word_rcv) = mpsc::unbounded_channel::<String>();

    // Start the background task that receives messages via the channel
    // and processes them. For simplicity, it takes strings and prints them.
    let bg_job = tokio::spawn({
        async move {
            // Message processing loop
            loop {
                tokio::select! {
                    // If a message is received, process it
                    word_resp = word_rcv.recv() => {
                        if let Some(word) = word_resp {
                            process_msg(word).await;
                        }
                    }
                    // If a shutdown signal is received, break the loop
                    _ = shutdown_rcv.recv() => {
                        println!("> Worker received shutdown command");
                        break;
                    }
                }
            }
            // Drain remaining messages in the channel after the shutdown signal
            while let Ok(word) = word_rcv.try_recv() {
                process_msg(word).await;
            }
            println!("> Worker is finished");
        }
    });

    let app = Router::new()
        .route("/enqueue/{word}", get(handle_request))
        .with_state(word_snd);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await.unwrap();

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();

    // After the server stops, notify the background process to finish up
    println!("> Sending shutdown command to workers");
    let _ = shutdown_snd.send(());

    let _ = bg_job.await;
}

// Endpoint that extracts a word from the URL path and sends it to the channel.
// The background task will then read and process this word.
async fn handle_request(
    Path(word): Path<String>,
    State(word_snd): State<mpsc::UnboundedSender<String>>,
) -> StatusCode {
    match word_snd.send(word) {
        Ok(_) => StatusCode::CREATED,
        Err(_) => StatusCode::INTERNAL_SERVER_ERROR,
    }
}

// Simulation of heavy work during message processing
async fn process_msg(w: String) {
    tokio::time::sleep(Duration::from_secs(1)).await;
    println!("Word: {w}");
}

async fn shutdown_signal() {
    let ctrl_c = signal::ctrl_c();

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("Cannot create SIGTERM handler")
            .recv()
            .await;
    };
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

Run the server (`cargo run`) and navigate to [http://localhost:8080/enqueue/hello](http://localhost:8080/enqueue/hello) in browser. You will see "Word: hello" printed in the console after a short delay.

Then terminate the server by pressing Ctrl+C in the console. The application will stop accepting new requests, signal the background worker, finish processing any "in-flight" words in the channel, and then exit gracefuly.


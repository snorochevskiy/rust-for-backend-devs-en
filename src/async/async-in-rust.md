# Asynchrony in Rust

When it comes to implementing user-space threads, two primary challenges must be addressed:

* How do we implement fibers so they are ergonomic to work with?
* How do we design an executor/scheduler for these fibers that is both flexible and high-performance?

Rust offers quite an elegant solution: the language and the standard library define a maximally flexible interface for asynchronous functions (which are essentially fibers), while the implementation of the executors used to run them is left to third-party developers. As a result, we have a unified interface for writing fibers while retaining the ability to choose one runtime or another depending on the specific needs of our application.

> [!NOTE]
> In documentation regarding user-space threads, you will often encounter the terms "executor" and "async runtime."
While they are sometimes used interchangeably, "async runtime" is a broader concept that includes the executor as just one of its components (albeit the most critical one).

## Async Functions

By declaring a function with the `async` keyword, we specify that the function is asynchronous and should be executed concurrently on some runtime.

```rust,noplayground
async fn function_name(arguments) -> ResultType {
    ...
}
```

For example:

```rust,noplayground
async fn get_1() -> i32 {
    1
}
```

Calling an asynchronous function does not return its value immediately; instead, it returns a fiber object. For instance, calling the `get_1()` function defined above creates a fiber object that returns `1` once it is evaluated.

```rust,noplayground
fn main() {
    let my_fiber = get_1();
}
```

To actually get the result from this fiber, we need to execute it on a runtime or an executor.

> [!NOTE]
> When reading documentation for async functions or runtimes, you will rarely encounter the word "fiber." You are much more likely to see the terms "async function" or "future." The former is used for obvious reasons — the `async` keyword. The latter refers to the trait that serves as the foundation for asynchronous execution in Rust, but we’ll dive into that later.\
> In any case, both "async function" and "future" are effectively synonyms for "fibers" in this context.

## Executing an Async Function { #async-fn-exec }

The Rust standard library provides the functionality to create fibers, but it does not include an executor implementation to run them.

The Rust ecosystem offers several high-performance asynchronous executors, but for now, we will use the most primitive executor available from the [futures](https://crates.io/crates/futures) crate.

First, let's add `futures` to your `Cargo.toml`:

```toml
[package]
name = "test_rust_async"
version = "0.1.0"
edition = "2024"

[dependencies]
futures = "0.3"
```

Now, here is `src/main.rs`:

```rust,edition2024
use futures::executor::block_on;

async fn get_1() -> i32 {
  1
}

fn main() {
  let my_fiber = get_1();
  let result = block_on(my_fiber);
  println!("{}", result);
}
```

The `block_on` function takes a fiber as an argument and executes it on an executor. This particular executor simply uses the current thread to run the fiber: it doesn't have a clever scheduler or a complex system of thread pools. However, this simple executor serves as an excellent demonstration of the fact that executors can be designed in many different ways.

We will explore more sophisticated executors in the following sections.

## Composition of Async Functions

We already know that calling an asynchronous function creates a fiber. But what if a fiber needs to consist of a sequence of async function calls?

For example, suppose we have two async functions:

```rust,noplayground
async fn load_number() -> i32 {
    1
}

async fn transform_number(a: i32) -> i32 {
    a + 1
}
```

And we want to create a fiber that first obtains a number by calling `load_number` and then transforms it using `transform_number`.

To compose asynchronous function calls in Rust, the **async/await** approach is used — one you may have already encountered in languages like JavaScript, C#, Python, or Swift.

```rust,noplayground
async fn my_flow() -> i32 {
    let number = load_number().await;
    let transformed = transform_number(number).await;
    transformed
}
```

Here, we are creating a third async function that combines calls to other async functions. Now, calling the `my_flow` function will return a fiber instance that contains sub-fibers spawned by the `load_number` and `transform_number` calls as its components.

It is important to note that the `await` call can <ins>only</ins> be made within the body of an async function.

> [!TIP]
> If you have encountered the `await` operator in other programming languages, you might have noticed that `await` is typically placed before the function call, whereas in Rust, it is placed after. This was designed to make it more convenient to chain async function calls:
> 
> ```rust,noplayground
> get_value().await
>     .call_method_1().await
>     .call_method_2().await
> ```

And now for the most important part: remember how we mentioned that a fiber contains points where the executor can suspend its execution? The `.await` calls are exactly those points. It is at these points that the executor can send the fiber to a wait queue, move it to another OS thread, and so on.

The internal mechanics of how `await` works will become clearer when we dive into the internal structure of a fiber, but for now, let's look at another example that is closer to the realities of backend development.

Suppose we have two separate services: one responsible for storing users and another for storing addresses. We want to implement functionality that returns information about a user and their address.

```rust,noplayground
// User information
struct User     { user_id: u64, name: String,   addr_id: u64 }
// Address information
struct Address  { addr_id: u64, location: String }
// Information about both the user and the address.
struct UserInfo { user: User,   addr: Address }

// Service that returns a user by their ID
async fn get_user_by_id(user_id: u64) -> User {
    // делает запрос к некому user сервису
}

// Service that returns an address by the address record ID
async fn get_address_by_id(addr_id: u64) -> Address {
    // делает запрос к некому address сервису
}

async fn get_user_with_address(user_id: u64) -> UserInfo {
  let user: User    = get_user_by_id(user_id).await;
  let addr: Address = get_address_by_id(user.addr_id).await;
  UserInfo { user, addr }
}
```

> [!TIP]
> <details>
> 
> <summary>A note for Java programmers</summary>
> 
> To understand how elegant composition via async/await is, let's consider what a similar function composition would look like if it were based on [CompletableFuture](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletableFuture.html):
> 
> ```java
> CompletableFuture<User> fetchUserById(Long userId) { ... }
> 
> CompletableFuture<Address> fetchAddressById(Long addrId) { … }
> 
> CompletableFuture<UserInfo> getUserInfo(Long userId) {
>   return fetchUserById(userId)
>     .thenCompose(user ->
>        fetchAddressById(user.getAddrId())
>          .thenApply(addr ->
>            new UserInfo(user, addr)
>          )
>     );
> }
> ```
> 
> You must agree that code using async-await is significantly easier to read.
> 
> </details>

## async Closures

A future can be created not only by calling an async function but also by using an async closure.

```rust,noplayground
let closure = async || { тело };
let fiber = closure();
```

For example:

```rust,edition2024
fn main() {
    let closure = async|| { 1 };
    let my_fiber = closure();
    let result = futures::executor::block_on(my_fiber);
    println!("{}", result); // 1
}
```

There are also async blocks. These are similar to a regular scope (a block of code in curly braces), but instead of returning a value immediately, they return a future.

```rust,noplayground
let fiber = async { тело };
```

For example:

```rust,edition2024
fn main() {
    let my_fiber = async { 1 };
    let result = futures::executor::block_on(my_fiber);
    println!("{}", result); // 1
}
```

## Fiber Anatomy — Future

Now, let’s dive into how fibers are structured internally.

If your code editor supports Rust LSP (which displays inferred types), the example from the [Исполнение async-функции](async-in-rust.md#async-fn-exec) section likely looks like this in your editor:

```rust,noplayground
use futures::executor::block_on;

async fn get_1() -> i32 {
    1
}

fn main() {
  let my_fiber: impl Future<Output = i32> = get_1();
  //            ^^^^^^^^^^^^^^^^^^^^^^^^
  let result: i32 = block_on(my_fiber);
  println!("{}", result);
}
```

As you can see, a fiber's type is something that implements the [Future](https://doc.rust-lang.org/std/future/trait.Future.html) trait. This `Future` trait is the primary interface for fibers.

```rust,noplayground
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

Indeed, all fibers in Rust must implement this specific generic trait, which contains only a single method. Let’s break it down.

The associated type `Output` is straightforward: it is the type of the result returned when the future (fiber) completes its work.

The `poll` method is used by the executor to drive the fiber toward completion. This method returns an enum wrapper called [Poll](https://doc.rust-lang.org/std/task/enum.Poll.html):

```rust,noplayground
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

`Ready(result)` signifies that the future has finished its work, and `Pending` indicates that the future is not yet ready to provide a result.

The [Context](https://doc.rust-lang.org/std/task/struct.Context.html) argument in the poll method is used to pass a reference to a [Waker](https://doc.rust-lang.org/std/task/struct.Waker.html) object into the future. The Waker, in turn, is used by the future to signal its executor that it is ready to make progress again.

How this mechanism works

When the executor calls poll to check the future's value, it passes a reference to a `Waker` via the `Context`.

* If the value is ready (or can be calculated immediately during the `poll` call), the value is returned wrapped in `Poll::Ready`.
* If the value cannot be calculated yet (the method returns `Poll::Pending`), the future saves a reference to the `Waker`.
* Later, once the internal task is actually finished, the future calls `Waker::wake()`. This "pokes" the executor, telling it: "Hey, you can call `poll` on me again now to get the result".

Ideally, the executor calls `poll` on a brand-new future object only once. If that call returns `Poll::Pending`, the executor moves the future to a "waiting" queue and doesn't touch it again until it receives a notification from that specific future via its `Waker`.

Let’s visualize a high-level interaction between the executor and futures. In this scenario:

* _Future 1_ represents a simple `async` function that doesn't depend on other async functions and performs no I/O.
* _Future 2_ initiates a non-blocking I/O operation using a separate thread that performs I/O in a loop (e.g., using `epoll`).

![](img/async-application_achitecture.svg)

Note that this specific I/O implementation isn't a strict requirement or a built-in standard. We are simply demonstrating one possible way it can be implemented.

> [!TIP]
> You might have noticed that in the `poll` method, the `self` argument is typed as `Pin<&mut Self>`. To keep things simple: [Pin](https://doc.rust-lang.org/std/pin/struct.Pin.html) is a wrapper that "pins" the future's memory to a specific location, preventing it from being moved.
> 
> `Pin` is rarely used outside of the context of futures, and you are unlikely to encounter it directly while building standard backend applications, so we won't dwell on it here.

## Writing Our Own Executor

By now, you should have a rough idea of how Futures work in Rust and the principles behind fiber executors.

However, if you feel like you're missing a "gut feeling" for how it all works under the hood, we will examine an example of a maximally primitive executor below. Be warned: the code is still quite complex. If you decide to skip straight to the next chapter, you won't miss any critical concepts for general usage.

Our executor example consists of two files:
* `src/my_executor.rs` — the module containing the executor implementation
* `src/main.rs` — the code to test the executor

File `src/my_executor.rs`:

```rust,noplayground
use std::{
    collections::HashSet, future::Future, pin::Pin,
    sync::{Arc, Mutex, atomic::{AtomicBool, AtomicU64, Ordering}, mpsc},
    task::{Context, Poll, Wake},
    thread::{self, sleep},
    time::Duration
};

// Alias for a "pinned" box containing a future.
// We must store futures as Pin<Box<dyn Future>> objects
// to be able to call poll, which requires Pin<&mut Self>
type BoxFuture = Pin<Box<dyn Future<Output = ()> + Send + 'static>>;

// A wrapper for a future generated by the compiler from an async function
struct SpawnedTask {
    id: u64,
    future: Mutex<Option<BoxFuture>>,
}

// A Future implementation that pauses (similar to thread::sleep)
// We will create this future in the main function.
pub struct Sleep {
    interval: Duration,
    is_ready: Arc<AtomicBool>,
}

impl Sleep {
    pub fn new(interval: Duration) -> Sleep {
        Sleep {
            interval, is_ready: Arc::new(AtomicBool::new(false))
        }
    }
}

impl Future for Sleep {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.is_ready.load(Ordering::SeqCst) {
            return Poll::Ready(());
        } else {
            let waker = cx.waker().clone();
            let ready_flag = self.is_ready.clone();
            let interval_to_sleep = self.interval.clone();
            // The most primitive implementation: spawn a new thread to wait
            thread::spawn(move || {
                sleep(interval_to_sleep);
                ready_flag.store(true, Ordering::SeqCst);
                // Notify the executor that the future has finished its work
                waker.wake();
            });
            Poll::Pending
        }
    }
}

// Interface for interacting with the executor
pub struct Executor {
    runtime: ExecutorRuntime,
    last_task_id: AtomicU64,
}

impl Executor {
    pub fn new() -> Executor {
        Executor {
            runtime: ExecutorRuntime::new(),
            last_task_id: AtomicU64::new(1),
        }
    }
    // Used to add an async function to the executor's queue
    pub fn spawn<F>(&self, fut: F) where F: Future<Output = ()> + Send + 'static {
        let task = Arc::new(SpawnedTask {
            id: self.last_task_id.fetch_add(1, Ordering::SeqCst),
            future: Mutex::new(Some(Box::pin(fut))),
        });
        let _ = self.runtime.task_producer.send(task);
    }

    // Starts evaluating futures (fibers) from the executor's queue
    pub fn exec_blocking(&mut self) {
        self.runtime.run();
    }
}

// Encapsulates the logic for the direct evaluation of futures
pub struct ExecutorRuntime {
    // Sender provided to other components (Executor and Tasks)
    // so they can add async functions to the queue
    task_producer: mpsc::Sender<Arc<SpawnedTask>>,
    // Receiver used by the runtime to pull the next async function
    task_queue: mpsc::Receiver<Arc<SpawnedTask>>,
    // Storage for futures that returned Poll::Pending on the first poll.
    // Necessary to avoid exiting before all futures are completed.
    task_pending: HashSet<u64>,
}

impl ExecutorRuntime {
    pub fn new() -> ExecutorRuntime {
        let (sender, receiver) = mpsc::channel::<Arc<SpawnedTask>>();
        ExecutorRuntime {
            task_producer: sender,
            task_queue: receiver,
            task_pending: HashSet::new(),
        }
    }

    // Start executing futures
    pub fn run(&mut self) {
        loop {
            match self.task_queue.recv_timeout(Duration::from_secs(1)) {
                Ok(task) => self.process_task(task),
                Err(_) =>
                    // If the queue is empty and no futures are 
                    // currently in progress, stop processing.
                    if self.task_pending.is_empty() {
                        break;
                    },
            }
        }
    }

    fn process_task(&mut self, task: Arc<SpawnedTask>) {
        let mut future_guard = task.future.lock().unwrap();
        // Extract the Pin<Box<dyn Future>> from the task because 
        // calling poll requires the object itself (by value), not a reference.
        let Some(mut fut) = future_guard.take() else {
            return; // already finished
        };

        // Create a Waker in case the future cannot complete immediately
        // and returns Poll::Pending.
        let spawned_task_waker = SpawnedTaskWaker {
            task: task.clone(),
            sender: self.task_producer.clone(),
        };
        let waker = Arc::new(spawned_task_waker).into();
        let mut cx = Context::from_waker(&waker);

        // Execute the future
        let poll_result = fut.as_mut().poll(&mut cx);

        match poll_result {
            Poll::Pending => {
                // Put the future back into the task since it will need 
                // to be processed again after the waker is triggered.
                *future_guard = Some(fut);
                // Record that the task with this ID is running in the background
                self.task_pending.insert(task.id);
            }
            Poll::Ready(()) => {
                // Remove the task ID from the background task list if it exists
                self.task_pending.remove(&task.id);
            }
        }
    }
}

// A simple Waker that just re-adds the task to the runtime queue
struct SpawnedTaskWaker {
    sender: mpsc::Sender<Arc<SpawnedTask>>,
    task: Arc<SpawnedTask>,
}

impl Wake for SpawnedTaskWaker {
    fn wake(self: Arc<Self>) {
        let _ = self.sender.send(self.task.clone());
    }
}
```

Example of using the executor in `main.rs`:

```rust,noplayground
mod my_executor;

use std::time::Duration;
use crate::my_executor::{Executor, Sleep};

async fn func_with_sleep() {
    println!("Async function: before sleep");
    Sleep::new(Duration::from_secs(1)).await;
    println!("Async function: after sleep");
}

async fn calc_5() -> i32 {
    5
}

fn main() {
    let mut ex = Executor::new();

    // Get the future (fiber)
    let fut = func_with_sleep();
    // Push the future into the executor's queue
    ex.spawn(fut);

    // Using an async closure instead of an async function
    ex.spawn(async {
        println!("Async closure: start");
        async fn inner() {
            println!("Async closure: inner function");
        }
        inner().await;

        let a = calc_5().await;
        println!("Async closure: async func call result {a}");
        println!("Async closure: end");
    });

    ex.exec_blocking();

    println!("All done");
}
```

This program will print:

```
Async function: before sleep
Async closure: start
Async closure: inner function
Async closure: async func call result 5
Async closure: end
Async function: after sleep
All done
```

If you are interested in seeing a more universal yet relatively simple executor, you can study the source code for the executor in the futures library:\
[https://github.com/rust-lang/futures-rs/tree/master/futures-executor](https://github.com/rust-lang/futures-rs/tree/master/futures-executor)

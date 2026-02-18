# Tokio

Now that we’ve familiarized ourselves with futures and executors, we can discuss how an efficient executor for backend applications should be designed. Backend applications are generally I/O-intensive (especially regarding networking), so an efficient executor must prioritize handling non-blocking I/O.

An executor should:

1. ...have a thread pool for executing tasks (fibers) that are strictly computational. This pool should not exceed the number of CPU cores.
2. ...provide its own non-blocking API, at least for networking operations.
3. ...have a single-threaded pool with the highest priority to handle non-blocking I/O operations via epoll, io_uring, IOCP, or kqueue.
4. ...maintain a thread pool of "unlimited" size for I/O operations that lack a non-blocking variant (e.g., standard DNS calls).

As it turns out, we have just roughly described the executor from the [Tokio](https://tokio.rs/) library.

**Tokio** is a high-performance asynchronous runtime for I/O-intensive applications. It includes both a multi-pool thread executor (including specialized pools for non-blocking I/O) and an extensive library of functions for asynchronous work with the network, file system, timers, and more.

> [!TIP]
> The Tokio library has very detailed documentation that is highly recommended reading: [https://tokio.rs/tokio/tutorial](https://tokio.rs/tokio/tutorial)

## Tokio I/O API

The first thing you’ll notice when studying Tokio is that the authors designed the API for asynchronous networking and file system operations to be very similar to the corresponding synchronous API in the standard library.

Take, for example, a simple program that reads the contents of a text file into a string.

First, let's add Tokio to `Cargo.toml`:

```toml
[package]
name = "test_rust"
version = "0.1.0"
edition = "2024"

[dependencies]
tokio = {version = "1", features = ["full"]}
```

Now, let's write the implementation using the synchronous API from the standard library:

```rust,noplayground
use std::fs::File;
use std::io::Read;

fn main() {
    let mut file = File::open("/etc/fstab")
        .unwrap();

    let mut contents = String::new();
    file.read_to_string(&mut contents)
        .unwrap();

    println!("{}", contents);
}
```

And here is the exact same logic using Tokio’s asynchronous API:

```rust,noplayground
use tokio::fs::File;
use tokio::io::AsyncReadExt;

#[tokio::main]
async fn main() {
    let mut file = File::open("/etc/fstab")
        .await
        .unwrap();

    let mut contents = String::new();
    file.read_to_string(&mut contents)
        .await
        .unwrap();

    println!("{}", contents);
}
```

As you can see, the code for interacting with the file is almost identical, except for the `.await` calls and the different signature of the `main` function.

This API design is one of the reasons why Tokio is so easy to learn.

## tokio::main

Now that we’ve seen what a Tokio program looks like, let's dive into the details.

The first thing to focus on is the `main` function. As you might have noticed, it became `async` and gained the `#[tokio::main]` annotation. There is no "magic" here: the entry point of any Rust program is the same non-async `main` function we've always known.

The `#[tokio::main]` procedural macro simply transforms this code:

```rust,noplayground
#[tokio::main]
async fn main() {
    // Async program code
}
```

into this:

```rust,noplayground
fn main() {
    let mut rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        // Async program code
    })
}
```

If we look back at the previous chapter and see how we [executed an async function on an executor](async-in-rust.md#async-fn-exec), we’ll see that a `block_on` call was present there as well, serving the same purpose.

Essentially, the Tokio executor works just like any other: the compiler transforms the async function into a future, and that future is then handed over to the executor for execution.

## Task

Just as you can create OS threads with traditional multithreading, Tokio allows you to manually create "fibers," which in Tokio terminology are called **tasks**.

Creating a task is performed using the [tokio::spawn](https://docs.rs/tokio/latest/tokio/task/fn.spawn.html) function, which is very similar to the [thread::spawn](https://doc.rust-lang.org/std/thread/fn.spawn.html) function used to create an OS thread..

The signature of the `tokio::spawn` function is as follows:

```rust,noplayground
pub fn spawn<F>(future: F) -> JoinHandle<F::Output> where
    F: Future + Send + 'static,
    F::Output: Send + 'static,
```

As you can see, this function takes a Future (an async block or the result of an async function call) as an argument.

For example:

```rust,edition2024
use tokio::task::{JoinError, JoinHandle};

async fn get_1() -> i32 {
    1
}

#[tokio::main]
async fn main() {
    // Creating a task from an async function call
    let t1: JoinHandle<i32> = tokio::spawn(get_1());
    let result1: Result<i32, JoinError> = t1.await;
    println!("{result1:?}"); // Ok(1)

    // Creating a task from an async block
    let t2: JoinHandle<i32> = tokio::spawn(async { 5 });
    let result2: Result<i32, JoinError> = t2.await;
    println!("{result2:?}"); // Ok(5)
}
```

From an API perspective, working with tasks is very similar to working with OS threads. The only difference is that OS threads are managed by the operating system, while tasks are managed by the Tokio runtime.

When we call `tokio::spawn`, a new task is created (which acts as a wrapper for the future generated by the compiler from an async function or block). This task is immediately placed into the Tokio executor's queue for execution. At some point, the Tokio scheduler will place this task onto one of the OS threads and attempt to execute it by calling the `poll` method on the future.

The `tokio::spawn` call returns a [tokio::task::JoinHandle](https://docs.rs/tokio/latest/tokio/task/struct.JoinHandle.html) object, which is very similar to the [std::thread::JoinHandle](https://doc.rust-lang.org/std/thread/struct.JoinHandle.html) returned by `thread::spawn`. It also allows you to wait for the task to complete and retrieve its result.

## task vs thread

We have discussed at length how user-space threads and non-blocking APIs are more efficient than OS threads and blocking APIs. Now that we have finally reached Tokio, let’s perform a quick comparison.

We will write a program that creates one million tasks, each of which simply sleeps for 1 second and then finishes its work.

```rust,noplayground
use std::time::Duration;

#[tokio::main]
async fn main() {
    let mut handles = Vec::new();
    for _ in 1..1000000 {
        let t = tokio::spawn(async {
            tokio::time::sleep(Duration::from_secs(1)).await;
        });
        handles.push(t);
    }
    for handle in handles {
        let _ = handle.await;
    }
}
```

Let's measure how long this program takes to run and the CPU load it generates. On Linux, for instance, you can run the program through the time utility (not to be confused with the built-in shell command). On Windows, you can use the PowerShell command [Measure-Command](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/measure-command).

```
$ cargo build

$ /bin/time cargo run
10.28user 4.07system 0:02.83elapsed 506%CPU (0avgtext+0avgdata 386308maxresident)k
0inputs+256outputs (0major+103536minor)pagefaults 0swaps
```

On the author's laptop (with an 8-core Ryzen 7840HS processor and 32 GB of RAM), the program finished in 2.83 seconds and used 386 MB of RAM at its peak.

Now, let's write the same program, but instead of Tokio tasks, we will use standard OS threads.

```rust,noplayground
use std::time::Duration;
use std::thread;

fn main() {
    let mut handles = Vec::new();
    for _ in 1..1000000 {
        let t = thread::spawn(|| {
            thread::sleep(Duration::from_secs(1));
        });
        handles.push(t);
    }
    for handle in handles {
        let _ = handle.join().unwrap();
    }
}
```

Let's run it and compare the results:

```
$ cargo build

$ /bin/time cargo run
thread '<unnamed>' (583640) panicked at library/std/src/sys/pal/unix/stack_overflow.rs:222:13:
failed to set up alternative stack guard page: Cannot allocate memory (os error 12)
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace

thread 'main' (91293) panicked at /home/stas/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/thread/mod.rs:729:29:
failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }
fatal runtime error: failed to initiate panic, error 5, aborting

Command terminated by signal 6
4.25user 71.05system 1:16.74elapsed 98%CPU (0avgtext+0avgdata 4219816maxresident)k
0inputs+0outputs (0major+1062256minor)pagefaults 0swaps
```

This version took 1 minute and 16 seconds and used 4.2 GB of RAM at its peak. As you can see, the difference is substantial.

You can also notice in the log that several attempts to create a thread failed.

> [!NOTE]
> There is a chance that when running on Linux, instead of taking a long time, the program might immediately terminate with an error like this:
> 
> ```
> $ cargo run
> thread 'main' (52703) panicked at /home/stas/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/> thread/mod.rs:729:29:
> failed to spawn thread: Os { code: 11, kind: WouldBlock, message: "Resource temporarily unavailable" }
> note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
> Command exited with non-zero status 101
> ```
> 
> This error indicates that the program attempted to exceed the maximum limit on the number of threads and was aborted.
> 
> If you are using GNOME Terminal with default settings, you will likely encounter this error. For a successful run, the author used the [Alacritty](https://alacritty.org/) terminal emulator—a cross-platform terminal written in Rust.


Comparing the results of these examples, we can confidently say that for certain tasks, Tokio is significantly more efficient than OS threads.

## Task Synchronization

Tokio aims to mirror the standard library not only in its filesystem and networking APIs but also in its synchronization primitives. That is why, for synchronizing data across different tasks, Tokio provides types like `Mutex`, `RwLock`, and `Barrier`, which closely resemble their counterparts used with OS threads.

For example, let's take a look at the Tokio mutex:

```rust,edition2024
use std::sync::Arc;
use tokio::sync::Mutex;

#[tokio::main]
async fn main() {
    let m: Arc<Mutex<i32>> = Arc::new(Mutex::new(1));

    let t = tokio::spawn({
        let m = m.clone();
        async move {
            let mut guard = m.lock().await;
            *guard = 2;
        }
    });
    let _ = t.await;

    println!("{m:?}");
}
```

As you can see, this example is very similar to the mutex example from the chapter on [multithreading](../dive-deeper/multithreading.md#mutex), with the primary difference being that the `lock()` call returns a future that must be `.await`ed.

> [!WARNING]
> Be careful: if you accidentally use a mutex designed for OS threads (from `std::sync`) inside a Tokio task, you may experience a significant drop in performance or even deadlocks in certain scenarios.

## Channels { #channels }

Tokio provides not only its own versions of synchronization primitives but also its own channels.

#### mpsc

[mpsc](https://docs.rs/tokio/latest/tokio/sync/mpsc/index.html) is the Tokio counterpart to the multiple producers, single consumer [channel](../dive-deeper/multithreading.md#channels) found in the standard library.

Let's look at an example:

```rust,edition2024
use std::time::Duration;
use tokio::sync::mpsc;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    // Create a channel
    let (snd, mut rcv) = mpsc::unbounded_channel::<i32>();

    // Send a number to the channel from a new task
    tokio::spawn({
        let snd = snd.clone();
        async move {
            let _ = snd.send(1);
        }
    });
    // Send a number to the channel from another new task
    tokio::spawn({
        let snd = snd.clone();
        async move {
            let _ = snd.send(2);
        }
    });

    // Read numbers from the channel in a loop and print them to the console.
    // If no new messages arrive within one second, terminate the loop.
    while let Some(msg) = tokio::select! {
        msg = rcv.recv()                    => msg,
        _   = sleep(Duration::from_secs(1)) => None,
    } {
        println!("Received: {msg}");
    }
}
```

Here, we first create an unbounded channel — `mpsc::unbounded_channel`.

Next, we start two tasks, each sending a single message to the channel.

At the end, we see an interesting block that has no direct equivalent in the standard library. We use the [tokio::select](https://tokio.rs/tokio/tutorial/select) macro, which takes several futures (two in our case) and returns the result of the one that completes first.

The syntax for using this macro is somewhat similar to a `match` expression:

```rust,noplayground
let variable = tokio::select! {
    pattern_1 = future_1 => expression_1,
    pattern_2 = future_2 => expression_2,
};
```

`select!` accepts multiple futures and waits for one of them to finish. When a future completes, its result is assigned to the corresponding pattern, and then the associated expression is executed. The result of this expression becomes the result of the entire `select!` block. Meanwhile, the results of the other futures are simply <ins>dropped</ins>.

Since the `tokio::time::sleep` function returns a future, it is often used in conjunction with the `select!` macro as a timeout for another operation. Thus, in the example above, we read messages from the channel with a 1-second timeout within the `select!` block.

#### oneshot

[oneshot](https://docs.rs/tokio/latest/tokio/sync/oneshot/index.html) is a channel designed for sending exactly one message.

The implementation is very elegant:

```rust,edition2024
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (snd, rcv) = oneshot::channel::<i32>(); // create the channel
    
    // The send method takes ownership of self, thereby consuming it.
    // Therefore, send can only be called once.
    let _ = snd.send(1);

    // The receiver for oneshot does not implement Clone, so it can
    // only be passed to a single location.

    // Since oneshot is designed to receive only one message,
    // we use .await directly instead of a receive() method.
    let r = rcv.await;

    println!("{r:?}"); // Ok(1)
}
```

#### broadcast

[broadcast](https://docs.rs/tokio/latest/tokio/sync/broadcast/index.html) is a multi-producer, multi-consumer channel where every message from any sender is delivered to all consumers.

```rust,edition2024
use std::{thread::sleep, time::Duration};
use tokio::sync::broadcast::{self, Receiver};

// Creates a receiver "living" in a separate task.
// The receiver gets one message, prints it, and terminates.
fn spawn_receiver(name: &'static str, rcv: &Receiver<i32>) {
    let mut rcv = rcv.resubscribe();
    tokio::spawn({
        async move {
            println!("{name} > msg: {:?}", rcv.recv().await);
        }
    });
}

#[tokio::main]
async fn main() {
    let (snd, rcv) = broadcast::channel::<i32>(100);

    spawn_receiver("rcv-1", &rcv);
    spawn_receiver("rcv-2", &rcv);
    spawn_receiver("rcv-3", &rcv);

    let _ = snd.send(5);

    sleep(Duration::from_secs(1));
}
```

The program will print:

```
rcv-2 > msg: Ok(5)
rcv-1 > msg: Ok(5)
rcv-3 > msg: Ok(5)
```

As you can see, each listener received its own copy of the message.

#### watch

The [watch](https://docs.rs/tokio/latest/tokio/sync/watch/index.html) channel allows sending messages from different tasks and reading them from multiple tasks as well. Crucially, the channel only retains the single latest value for each reader.

This type of channel is typically used to notify listeners about state changes, keeping them synchronized with the current state.

```rust,edition2024
use tokio::sync::watch;

#[tokio::main]
async fn main() {
    let (snd, rcv) = watch::channel::<i32>(0);

    // has_changed() indicates whether messages have arrived 
    // that haven't been seen by THIS specific receiver yet.
    println!("Has new messages: {}", rcv.has_changed().unwrap());

    let _ = snd.send(1);
    let _ = snd.send(2);
    let _ = snd.send(3);

    {
        let mut rcv1 = rcv.clone();
        println!("Receiver 1 changed: {}", rcv1.has_changed().unwrap());
        {
            // borrow_and_update() returns a reference to the latest received value
            // and marks this value as "seen" for this receiver.
            let guard = rcv1.borrow_and_update();
            println!("Receiver 1 value: {:?}", *guard);
        }
        println!("Receiver 1 changed: {}", rcv1.has_changed().unwrap());
    }

    {
        let mut rcv2 = rcv.clone();
        println!("Receiver 2 changed: {}", rcv2.has_changed().unwrap());
        {
            let guard = rcv2.borrow_and_update();
            println!("Receiver 2 value: {:?}", *guard);
        }
        println!("Receiver 2 changed: {}", rcv2.has_changed().unwrap());
    }
}
```

The program will output:

```
Has new messages: false
Receiver 1 changed: true
Receiver 1 value: 3
Receiver 1 changed: false
Receiver 2 changed: true
Receiver 2 value: 3
Receiver 2 changed: false
```

#### async_channel

In backend applications, you often need to implement functionality where request handlers generate tasks for resource-intensive work (such as image processing), which are then handled by a pool of workers.

![](img/tokio_async_channel.svg)

This seems like a perfect scenario for channels. However, the problem is that Tokio does not provide a built-in channel type that allows multiple tasks to send messages and multiple tasks to pull messages. In other words, there is no [MPMC](https://doc.rust-lang.org/std/sync/mpmc/index.html) (Multi-Producer Multi-Consumer) analog in Tokio itself.

Fortunately, there is a third-party library called [async-channel](https://crates.io/crates/async-channel) that works seamlessly with Tokio and provides exactly this type of channel.

```rust,noplayground
use async_channel::{Receiver, Sender};
use std::time::Duration;
use tokio::{task::JoinHandle, time::sleep};

// Creates a task that sends a specified value into the channel
fn make_producer(snd: &Sender<i32>, value: i32) {
    let snd = snd.clone();
    tokio::spawn(async move {
        let _ = snd.send(value).await;
    });
}

// Creates a worker task that reads messages from the channel in a loop and prints them.
// If no new messages arrive within one second, the worker terminates.
fn make_worker(rcv: &Receiver<i32>, name: &'static str) -> JoinHandle<()> {
    let rcv = rcv.clone();
    tokio::spawn(async move {
        loop {
            tokio::select! {
                msg = rcv.recv() => {
                    println!("{name} > received: {:?}", msg.unwrap());
                    sleep(Duration::from_millis(100)).await;
                }
                _ = sleep(Duration::from_secs(1)) => break,
            }
        }
    })
}

#[tokio::main]
async fn main() {
    let (snd, rcv) = async_channel::unbounded::<i32>();
    make_producer(&snd, 1);
    make_producer(&snd, 2);
    make_producer(&snd, 3);
    make_producer(&snd, 4);

    let t1 = make_worker(&rcv, "worker-1");
    let t2 = make_worker(&rcv, "worker-2");
    let _ = t1.await;
    let _ = t2.await;
}
```

Программа напечатает:

```
worker-2 > received: 1
worker-1 > received: 2
worker-1 > received: 4
worker-2 > received: 3
```

## LocalSet

The Tokio executor operates in such a way that a single task can be partially executed on one OS thread, then suspended and placed in a wait queue for an I/O operation, and later resumed on a completely different OS thread.

This is exactly why data captured by a task must implement `Send`: because a task can always be moved to another thread for execution.

However, Tokio provides a mechanism that allows you to specify that a task must run on the same OS thread and never be moved to others. This mechanism is called [LocalSet](https://docs.rs/tokio/latest/tokio/task/struct.LocalSet.html).

```rust,noplayground
let local_set = LocalSet::new();
// Этот таск будет выполнен на том же ОС потоке,
// который выполняет файбер, запустивший таск
// посредством LocalSet
local_set.spawn(async { тело таска });
```

Let’s consider an example where we increment a counter from multiple tasks.

When working with standard tasks, we would have to wrap the counter in a `Mutex` and then wrap that mutex in an `Arc` (which implements `Send`). However, `LocalSet` allows us to use types that do not implement `Send`, meaning we can use a simple `Rc` instead of `Arc`.

```rust,edition2024
use std::{rc::Rc, time::Duration};

use tokio::{sync::Mutex, task::LocalSet, time::sleep};

#[tokio::main]
async fn main() {
    let counter = Rc::new(Mutex::new(0)); // Rc does not implement Send
    let local_set = LocalSet::new();
    for _ in 0..100 {
        let counter = counter.clone();
        local_set.spawn_local(async move { // spawn a task on the current OS thread
            // Sleep for 1 second
            sleep(Duration::from_secs(1)).await;
            *counter.lock().await += 1;
        });
    }
    local_set.await;
    println!("{}", *counter.lock().await);
}
```

To verify that the tasks are running concurrently, let's run the program using the time utility:

```
$ /bin/time cargo run
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.04s
     Running `target/debug/test_rust`
100
0.09user 0.05system 0:01.14elapsed 13%CPU (0avgtext+0avgdata 30524maxresident)k
0inputs+0outputs (0major+8035minor)pagefaults 0swaps
```

As we can see, despite 100 tasks sleeping for 1 second each, the total execution time was approximately 1.14 seconds. Therefore, the tasks were indeed sleeping concurrently.

## Task local

When dealing with OS threads, we can use thread-local storage — [Thread local](../dive-deeper/multithreading.md#thread-local-storage). A similar mechanism exists for Tokio tasks: **Task Local**. It allows you to create a "global" variable that is unique to each <ins>task</ins>.

Task-local variables are declared using the [task_local](https://docs.rs/tokio/latest/tokio/macro.task_local.html).

```rust,noplayground
tokio::task_local! {
    pub static VARIABLE: Type;
}
```

Once declared, you can create a task-local scope with a specific value.

```rust,noplayground
VARIABLE.scope(value, async {
   // Code in this scope, as well as all functions called from this scope,
   // have access to the task-local variable as if it were a global variable
   // of type LocalKey<Type>
   my_func()
});

fn my_func() {
    let v = VARIABLE.get();
    // ...
}
```

All functions called within this scope will have access to this variable as if it were global. The type of this global variable is [LocalKey\<T>](https://docs.rs/tokio/latest/tokio/task/struct.LocalKey.html), where `T` is the type you declared in the `task_local!` macro.

If you declared your task-local variable with a type that implements `Copy` (for example, `i32`), you can use the `get()` method on the `LocalKey` object to simply retrieve a copy of the value.

For example:

```rust,edition2024
tokio::task_local! {
    pub static NUM: i32;
}

fn print_num() {
    println!("NUM = {}", NUM.get());
}

#[tokio::main]
async fn main() {
    NUM.scope(1, async { print_num() }).await;
    NUM.scope(2, async { print_num() }).await;
}
```

The program will print:

```
NUM = 1
NUM = 2
```

If you need to get a reference to the value stored in the task-local variable, you can use the [with](https://docs.rs/tokio/latest/tokio/task/struct.LocalKey.html#method.with) method:

```rust,noplayground
pub fn with<F, R>(&'static self, f: F) -> R where F: FnOnce(&T) -> R
```

This method takes and executes a closure that expects a reference to the task-local variable as an argument.

```rust,edition2024
tokio::task_local! {
    pub static NAME: String;
}

fn print_name() {
    NAME.with(|name| println!("NAME = {name}"))
}

#[tokio::main]
async fn main() {
    NAME.scope("A".to_string(), async { print_name() }).await;

    NAME.scope("B".to_string(), async { print_name() }).await;
}
```

***

What are the practical applications of task-local variables?

When talking about backend applications, we often encounter the concept of a "cross-cutting concern." This is a type of functionality that permeates the entire program logic and, therefore, cannot be easily isolated into a single separate module. Classic examples of cross-cutting concerns include authentication, authorization, and logging.

For instance, an application might have many different features that should only execute if the request was initiated by a user authorized to perform that specific action. This raises a question: how do we pass information about the user and their permissions throughout the call stack?

The First Approach: Introduce an argument to every function involved in processing the request to store user information. This argument would have to be added to all functions—both those that directly check user info and those that simply call other functions requiring that info. As you can imagine, this isn't exactly the most elegant solution.

The Second Approach: At the very beginning of the request processing, store the user information in a task-local variable. From that point on, any part of the code that needs it can simply read it from the task-local storage.

---

For our example, let's imagine we are writing an HTTP server. This server provides an endpoint, `/orders/submit`, which allows users authorized with the `CUSTOMER` role to submit orders for processing.

The expected flow is as follows: a user sends an HTTP request to `/orders/submit` with the order text in the request body. Authentication is handled by passing a session ID through an HTTP header.

We will write a simple router that checks the URL path. If the path matches `/orders/submit`, the router calls the corresponding endpoint handler. However, this handler is wrapped in a special <ins>middleware</ins> that extracts the user's session ID from the request headers, looks up the user's role by that ID, and then places all this information into a task-local variable. Subsequently, the request handler executes, retrieving the user's role from the task-local storage to determine if they have the right to submit orders.

![](img/tokio-task_local_example.svg)

If we add more endpoints later, we can reuse this middleware to populate the task-local storage with user information for them as well.

For simplicity, we won't write any actual networking code; instead, we'll manually create a request object and pass it to the router.

```rust,edition2024
use std::{cell::RefCell, collections::HashMap};

tokio::task_local! {
    pub static USER: RefCell<UserInfo>;
}

#[derive(PartialEq, Eq)]
enum Role {
    CUSTOMER, GUEST,
}

struct UserInfo {
    id: String, // User ID (taken from the request)
    role: Role, // Role (determined on the server by ID)
}

// Represents an HTTP request from a user
struct Request {
    path: String, // URL path from the request
    headers: HashMap<String, String>, // HTTP headers
    payload: String, // Request body
}

// Middleware that runs before the handler to 
// push user info and permissions into Task Local storage.
// * request - the request object
// * request_handler - the handler to be executed 
//   after the user data is prepared
async fn auth_middleware<F, Fut>(
    request: Request,
    request_handler: F
) -> Result<String, String>
    where 
        F : Fn(Request) -> Fut,
        Fut: Future<Output = Result<String, String>>
{
    // Extract the session ID from the "USER-ID" header.
    // If the header is missing, stop processing immediately.
    let Some(user_id) = request.headers.get("USER-ID") else {
        return Err("No USER-ID header".to_string());
    };
    // Retrieve the role associated with the session ID from storage.
    let Some(role) = find_user_role(user_id.as_str()) else {
        return Err(format!("No role for user with ID={user_id}"));
    };
    // Place user information into Task Local storage 
    // and call the request handler
    USER.scope(
        RefCell::new(UserInfo { id: user_id.to_string(), role }),
        async move { request_handler(request).await },
    ).await
}

fn find_user_role(id: &str) -> Option<Role> {
    // Simulating a database lookup
    match id  {
        "ID_1111" => Some(Role::CUSTOMER),
        "ID_2222" => Some(Role::GUEST),
        _         => None,
    }
}

// Handler for requests with the path /orders/submit
async fn handle_submit_order(request: Request) -> Result<String, String> {
    // This handler simply calls a business logic function
    submit_user_order(request.payload).await
}

// Business logic for processing a user's order
async fn submit_user_order(order: String) -> Result<String, String> {
    // This functionality is only available to users with the CUSTOMER role
    USER.with(|u| {
        if u.borrow().role != Role::CUSTOMER {
            Err(format!("User {} is not authorized", u.borrow().id))
        } else {
            Ok(format!("Order submitted: {order}"))
        }
    })
    

}

// Function that determines which handler should 
// process the request based on the URL path.
async fn serve_request(request: Request) -> Result<String, String> {
    // For this simple example, we only expect one URL path
    match request.path.as_str() {
        "/orders/submit" => auth_middleware(request, handle_submit_order).await,
        _ => Err("Unexpected path".to_string())
    }
}

#[tokio::main]
async fn main() {
    // Request without a session ID
    {
        let request = Request {
            path: "/orders/submit".to_string(),
            headers: HashMap::new(),
            payload: "Test payload 1".to_string()
        };
        let response = serve_request(request).await;
        println!("Response: {response:?}");
    }

    // Request with a CUSTOMER session ID
    {
        let headers = HashMap::from([
            ("USER-ID".to_string(), "ID_1111".to_string())
        ]);
        let request = Request {
            path: "/orders/submit".to_string(),
            headers,
            payload: "Test payload 2".to_string()
        };
        let response = serve_request(request).await;
        println!("Response: {response:?}");
    }

    // Request with a GUEST session ID
    {
        let headers = HashMap::from([
            ("USER-ID".to_string(), "ID_2222".to_string())
        ]);
        let request = Request {
            path: "/orders/submit".to_string(),
            headers,
            payload: "Test payload 3".to_string()
        };
        let response = serve_request(request).await;
        println!("Response: {response:?}");
    }
}
```

The program outputs:

```
Response: Err("No USER-ID header")
Response: Ok("Order submitted: Test payload 2")
Response: Err("User ID_2222 is not authorized")
```

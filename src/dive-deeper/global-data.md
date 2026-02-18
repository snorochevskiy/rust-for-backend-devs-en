# Global Data

Now that we have tackled multithreading and understand the challenges of synchronizing data access, we are ready to look at working with global data.

As we already know, there are two types of global data in Rust: constants and static data.

Constants are extremely simple to use because they:

* Are available from the very first moment the program starts (they have a `'static` lifetime).
* Are immutable, which means no reference restrictions apply to them.

However, constants have significant limitations:

* The value of a constant must be computed at compile time, meaning a constant cannot be a data type that uses the heap (e.g., `HashMap`).
* Sometimes it is necessary to change the value of global data during program execution, but constants do not allow this.

## Global Static Variables

If you need to store data in a global variable that cannot be initialized at compile time, or if you need the global variable to be mutable, you will have to use a global static variable.

We have already seen [static variables inside functions](../rust-basics/functions.md#static-variables). Global static variables differ only in that they are accessible not just within a single function, but from the entire program. Everything else is the same.

A static global variable, like a local one, is declared using the `static` keyword.

```rust,noplayground
static VAR1: i32 = 1;
static mut VAR2: i32 = 2;
```

As a reminder:

* Static variable names are written in ALL_CAPS.
* Static variables can be either mutable or immutable.
* The variable type is mandatory, even if the compiler could unambiguously infer it from the value.

Reading and writing to a static mutable variable is considered a potentially dangerous operation because this variable can be used simultaneously from different threads. This is why reading or changing the value of a static mutable variable is only allowed inside an `unsafe` block.

```rust
static mut VAR1: i32 = 0;

fn inc_var() {
    unsafe {
        VAR1 += 1;
    }
}

fn get_var() -> i32 {
    unsafe {
        VAR1
    }
}

fn main() {
    println!("{}", get_var()); // 0
    inc_var();
    println!("{}", get_var()); // 1
}
```

At this stage, it already becomes clear that using mutable static variables without some kind of synchronization mechanism is a bad idea.

## Synchronizing Global Variables

From the chapter on multithreading, we already know that a shared variable can be synchronized using a mutex or an `RwLock`This works not only for local variables captured by a closure but also for global ones.

Let's look at an example of synchronizing access to a global static variable using a mutex.

```rust
use std::{sync::Mutex, thread, time::Duration};

static COUNTER: Mutex<u64> = Mutex::new(0);

fn main() {
    thread::scope(|s| {
        s.spawn(|| {
            for i in 0..100 {
                let mut guard = COUNTER.lock().unwrap();
                let curr_val = *guard;
                thread::sleep(Duration::from_millis(10));
                *guard = curr_val + 1;
            }
        });
        s.spawn(|| {
            for i in 0..100 {
                let mut guard = COUNTER.lock().unwrap();
                let curr_val = *guard;
                thread::sleep(Duration::from_millis(10));
                *guard = curr_val + 1;
            }
        });
    });

    println!("{}", *COUNTER.lock().unwrap()); // 200
}
```

As you can see, threads work with the globally declared `COUNTER` mutex exactly as if it were created within the body of the `main` function.

## LazyLock

In the previous example, we synchronized access to a global variable of a type that does not use the heap — `i32`. However, if we attempt to create a global hash map in the same manner, we will encounter a compilation error:

```rust,compile_fail
use std::{collections::HashMap, sync::Mutex};

static m: Mutex<HashMap<String, i32>> = Mutex::new(HashMap::new());

fn main() {}
```

error:

```
error[E0015]: cannot call non-const associated function `HashMap::<String, i32>::new` in statics
 --> src/main.rs:3:52
  |
3 | static m: Mutex<HashMap<String, i32>> = Mutex::new(HashMap::new());
  |                                                    ^^^^^^^^^^^^^^
  |
  = note: calls in statics are limited to constant functions, tuple structs and tuple variants
  = note: consider wrapping this expression in `std::sync::LazyLock::new(|| ...)`
```

The compiler message explains that the initial value of a static variable can only be constructed using a [constant function](../rust-basics/functions.md#const-functions). While `Mutex::new` is a const function, `HashMap::new` is not.

The compiler also graciously suggests solving this problem by using [LazyLock](https://doc.rust-lang.org/std/sync/struct.LazyLock.html).

`LazyLock` is a wrapper that allows you to defer value initialization until it is requested for the first time. In other words, it moves the actual initialization from compile time to runtime.

A `LazyLock` value is created using the constant function `LazyLock::new`, which makes it suitable for initializing static global variables.

```rust,noplayground
pub const fn new(f: F) -> LazyLock<T, F>
```

It takes a closure as an argument, which should construct the required object. This closure will be invoked upon the first access to the `LazyLock` value.

```rust
use std::{ collections::HashMap, sync::{LazyLock, Mutex} };

// Create a hash map synchronized by a Mutex 
// and initialized via LazyLock
static M: LazyLock<Mutex<HashMap<String, i32>>> = LazyLock::new(
    || Mutex::new(HashMap::new())
);

fn main() {
    { // Add an element to the hash map
        let mut guard = M.lock().unwrap();
        guard.insert("one".to_string(), 1);
    }
    { // Read values from the hash map
        let guard = M.lock().unwrap();
        println!("{:?}", *guard); // {"one": 1}
    }
}
```

`LazyLock` implements the `Deref` trait, which allows you to work with it transparently, as if you were interacting with its inner content directly.

> [!NOTE]
> If the triple-decker generic `LazyLock<Mutex<HashMap<String, i32>>>` looks intimidating at first glance, don't worry: you will quickly grow to appreciate the explicitness of such declarations.

To ensure synchronization, `LazyLock` utilizes double-checked locking, making it safe to use in a multi-threaded environment.

## OnceLock

`LazyLock` addresses the issue of initializing global variables of "complex" types. However, it is not suitable for cases where a global variable might need to be initialized from several different locations in the code—for instance, if different functions must be called based on specific conditions to initialize the variable differently. In these scenarios, [OnceLock](https://doc.rust-lang.org/std/sync/struct.OnceLock.html) is the solution.

`OnceLock` serves a similar purpose to `LazyLock`, but instead of initializing upon the first access, it is initialized via an explicit call to the `set` method. The `set` method assigns a value to the `OnceLock` only if it is the very first time `set` is called. Otherwise, the call is ignored (and returns an error).

```rust
use std::sync::{Mutex, OnceLock};

static O: OnceLock<Mutex<String>> = OnceLock::new();

fn main() {
    { // Initialize the OnceLock value
        let r = O.set(Mutex::new("1".to_string()));
        if let Err(e) = r {
            eprintln!("Cannot init with: {e:?}");
        }
    }
    { // Attempting to re-initialize the OnceLock
        let r = O.set(Mutex::new("2".to_string()));
        if let Err(e) = r {
            eprintln!("Cannot init with: {e:?}");
        }
    }
    {
        let mutex = O.get().unwrap();
        let guard = mutex.lock().unwrap();
        println!("OnceLock = {:?}", *guard);
    }
}
```

This program will print:

```
Cannot init with: Mutex { data: "2", poisoned: false, .. }
OnceLock = "1"
```

Note: Calling `.unwrap()` on the result of `get()` on an uninitialized `OnceLock` object will result in a panic, as `get()` returns `None` if the lock is empty.

`OnceLock` also provides a `get_or_init` method, which accepts a closure to generate a value. If the `OnceLock` has already been initialized, `get_or_init` returns a reference to the existing value; otherwise, it executes the closure to initialize it and then returns a reference to the newly created value.

```rust
use std::sync::{Mutex, OnceLock};

static O: OnceLock<Mutex<String>> = OnceLock::new();

fn main() {
    {
        let mutex = O.get_or_init(|| Mutex::new("default".to_string()));
        let guard = mutex.lock().unwrap();
        println!("OnceLock = {:?}", *guard);
    }
}
```

## Example: Storing User Sessions

Now let's consider an example that frequently arises when writing backend applications: storing user sessions.

Suppose user data is represented by some structure `UserSession` (the specific implementation is not important for our purposes), and each session is identified by a unique string key, such as a UUID.

A `HashMap<String, UserSession>` is a convenient foundation for such a session store. For instance, this hash map could belong to a singleton object (e.g., a `SessionManager` struct) or be a global static variable. To further explore the topic, let's use a static variable.

We already know that we will need to synchronize access to this hash map. New sessions are created far less frequently than existing ones are read, so in our case, an `RwLock` will be more efficient than a `Mutex`. We also know that we will need `LazyLock`.

```rust,noplayground
static SESSIONS: LazyLock<RwLock<HashMap<String, UserSession>>> =
    LazyLock::new(|| RwLock::new(HashMap::new()));
```

Backend applications can handle thousands of concurrent user requests, and session data might be required multiple times during the processing of each request. Consequently, such a session store could become a "bottleneck." We need to ensure that at the very beginning of request processing, the handler obtains the user session object and then works with it without holding onto the entire `RwLock` (even for reading). We can achieve this by wrapping the session object in an `Arc`.

```rust,noplayground
static SESSIONS: LazyLock<RwLock<HashMap<String, Arc<UserSession>>>> =
    LazyLock::new(|| RwLock::new(HashMap::new()));
```

Now the handler can obtain a smart pointer to the required session object and work with it.

```rust,noplayground
fn serve_request(r: Request) -> Response {
    let session: Arc<UserSession> = {
        // Acquire the hash table for reading
        let hash_table = SESSIONS.read().unwrap();
        if let Some(session_ref) = hash_table.get(r.get_session_id()) {
            session_ref.clone(); // клонируем Arc
        } else {
            // Session with this ID not found in the hash table
            return Response::error_301();
        }
    };
    // Now we can freely work with the session without blocking the RwLock
    Response::ok_200()
}
```

Now we only acquire the `RwLock` for a short period to retrieve the session object from the hash table, after which we immediately release it. However, a new problem has arisen: Arc does not provide interior mutability, so it doesn't allow changing its contents, yet we want the request handler to be able to modify its corresponding session object.

This restriction from `Arc` is perfectly justified: theoretically, we could be processing multiple requests from the same user simultaneously, and we don't want these handlers to corrupt session data due to unsynchronized access.

Fortunately, we already know the solution to this problem: we need to place a `Mutex` or `RwLock` inside the `Arc`. Let's assume it's better for request handlers to have exclusive access to session data, so we'll use a `Mutex`.

```rust,noplayground
static SESSIONS: LazyLock<RwLock<HashMap<String, Arc<Mutex<UserSession>>>>> =
    LazyLock::new(|| RwLock::new(HashMap::new()));
```

Now our user request handler might look like this:

```rust,noplayground
fn serve_request(r: Request) -> Response {
    let session_mutex: Arc<Mutex<UserSession>> = {
        let hash_table = SESSIONS.read().unwrap();
        if let Some(session_ref) = hash_table.get(r.get_session_id()) {
            session_ref.clone();
        } else {
            return Response::error_301();
        }
    };
    // Extract some value from the session
    let mut some_value = {
        let guard = session_mutex.lock().unwrap();
        (*guard).some_field.clone();
    };
    // Some calculations involving some_value
    {
        // Update the session data
        let guard = session_mutex.lock().unwrap();
        // It's a good idea here to check if 'some_field' was changed 
        // by another concurrent handler that might exist
        (*guard).some_field = some_value;
    }
    Response::ok_200()
}
```

Of course, we are using `unwrap` on `RwLock` and `Mutex` objects solely for the sake of the example. In a real-world application, you must handle potential errors properly.


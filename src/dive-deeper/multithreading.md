# Multithreading

When it comes to concurrent programming, Rust provides:

* Traditional multithreading based on threads managed by the operating system.
* Asynchronous programming, which allows functionality to run on an async runtime operating in user space.

In this chapter, we will focus on working with OS threads.

## Creating a Thread

To create a new thread, the [std::thread::spawn](https://doc.rust-lang.org/std/thread/fn.spawn.html) function is used:

```rust,noplayground
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T + Send + 'static,
    T: Send + 'static,
```

The spawn function takes an `FnOnce` closure as an argument, which is then executed on a new thread. Naturally, you can pass not only FnOnce, but also `FnMut`, `Fn`, and `fn()`. We will discuss the `Send` trait a bit later.

The `spawn` function returns a [JoinHandle](https://doc.rust-lang.org/std/thread/struct.JoinHandle.html) object, which allows you to:

* Get information about the thread.
* Wait for the thread to finish
* Retrieve the result returned by the thread's function.

Let's look at a simple example: we will create two threads, each of which simply returns a number. The threads will be created using anonymous functions.

```rust
use std::thread::{self, JoinHandle};

fn main() {
    let t1: JoinHandle<i32> = thread::spawn(||{ 1 });
    let t2: JoinHandle<i32> = thread::spawn(||{ 2 });

    let sum = t1.join().unwrap() + t2.join().unwrap();

    println!("{sum}"); // 3
}
```

A thread can also be created from a regular function:

```rust
use std::thread;

fn print_nums() {
    let thread_id = thread::current().id(); // Current thread ID
    for i in 1 .. 5 {
        println!("thread: {thread_id:?}, num: {i}");
        thread::sleep(std::time::Duration::from_millis(100));
    }
}

fn main() {
    let t1 = thread::spawn(print_nums);
    let t2 = thread::spawn(print_nums);
    let _ = t1.join();
    let _ = t2.join();
}
```

This program prints:

```
thread: ThreadId(2), num: 1
thread: ThreadId(3), num: 1
thread: ThreadId(2), num: 2
thread: ThreadId(3), num: 2
thread: ThreadId(2), num: 3
thread: ThreadId(3), num: 3
thread: ThreadId(2), num: 4
thread: ThreadId(3), num: 4
```

---

In reality, threads are launched using the [std::thread::Builder](https://doc.rust-lang.org/std/thread/struct.Builder.html), which allows for more granular configuration of the created thread. The `std::thread::spawn` function is simply a shorthand that calls `std::thread::Builder::new().spawn(closure)` with default parameters. However, if you need to specify parameters such as a custom thread name or stack size, you can do so by creating the thread directly via the builder.

```rust
fn main() {
    let t = std::thread::Builder::new()
        .name("my_thread".to_string())
        .stack_size(8192)
        .spawn(|| {
            println!("thread name: {:?}", std::thread::current().name()) 
        })
        .unwrap();
    let _ = t.join();
}
```

## The Send Trait

It’s time to discuss moving objects between threads and the role of the [Send](https://doc.rust-lang.org/std/marker/trait.Send.html) trait.

Let's look at the following example:

```rust
use std::thread;

fn main() {
    let user = "John Doe".to_string();
    let t1 = thread::spawn(move || {
        println!("{}", user);
    });
    let _ = t1.join();
}
```

Everything is straightforward here: we create a string object in the main thread and then use that object inside a spawned thread.

We are already familiar with the `move` keyword: it signifies that if a value from the outer context is used by the closure (including use by reference), ownership of that object must be moved to the closure.

Now, let's modify this program by wrapping the string in an `Rc` smart pointer.

```rust,compile_fail
use std::{rc::Rc, thread};

fn main() {
    let user = Rc::new("John Doe".to_string());
    let t1 = thread::spawn(move || {
        println!("{}", user);
    });
    let _ = t1.join();
}
```

Compilation will fail with an error:

```
error[E0277]: `Rc<String>` cannot be sent between threads safely
   --> src/main.rs:5:28
    |
5   |       let t1 = thread::spawn(move || {
    |                ------------- ^------
    |                |             |
    |  ______________|_____________within this `{closure@src/main.rs:5:28: 5:35}`
    | |              |
    | |              required by a bound introduced by this call
6   | |         println!("{}", user);
7   | |     });
    | |_____^ `Rc<String>` cannot be sent between threads safely
    |
    = help: within `{closure@src/main.rs:5:28: 5:35}`,
      the trait `Send` is not implemented for `Rc<String>`
```

The error states that an `Rc` object cannot be sent between threads because the `Rc` type does not implement the `Send` trait.

`Send` is a <ins>marker</ins> trait indicating that an object of this type can be safely transferred between threads.

Objects of the `Rc` type are unsafe to transfer between threads. To understand why, let's imagine the following scenario:

1. In the main thread, we create an Rc object. As you may recall, [Rc consists of two fields](../rust-basics/smart-pointers.md#rc): a pointer to the data on the heap and a pointer to the counter (the number of owners), which is also located on the heap.
2. We clone the `Rc`, resulting in two `Rc` objects pointing to the same data on the heap.
3. We pass the second `Rc` object to another thread.
4. Simultaneously, in both the main thread and the second thread, we clone the `Rc` objects.

Since the counter on the heap is not synchronized for multi-threaded access, there is a risk that simultaneous increments from different threads will result in an incorrect value being written. This makes it easy to see why `Rc` is unsafe to transfer between threads, and consequently, why the `Send` trait is not implemented for `Rc`.

The `Send` trait is automatically implemented by the compiler for any type as long as it does not contain fields that are not `Send` (for example, fields of type `PhantomData` or `*mut T`).

To demonstrate this, let's create our own type that is not `Send`:

```rust,compile_fail
struct User {
    name: String,
    ptr: *mut u32, // *mut T pointer is not Send
}

// This function can only be called if T implements Send
fn prove_send<T: Send>() {}

fn main() {
    prove_send::<User>(); // Error: User is not Send
}
```

If we still need the ability to transfer `User` objects between threads (effectively making it `Send`), we can explicitly implement the `Send` trait like this:

```rust
struct User {
    name: String,
    ptr: *mut u32, // *mut T pointer is not Send
}

unsafe impl Send for User { } // Explicitly implement Send

fn prove_send<T: Send>() {}

fn main() {
    prove_send::<User>(); // Works without errors
}
```

Of course, `using` an unsafe trait implementation explicitly indicates that the developer now carries full responsibility for the correct handling of pointers. Consequently, we must manually ensure access to the `ptr` field is synchronized.

## Sync

Another important trait in the context of multithreading is [Sync](https://doc.rust-lang.org/std/marker/trait.Sync.html). This is also a marker trait, and it is automatically implemented by the compiler for any type if an immutable reference to a value of that type can be safely used across multiple threads.

Formally speaking: `T` is `Sync` if an immutable reference `&T` is `Send`.

Most standard types implement `Sync`. As with `Send`, the exceptions are types that encapsulate a pointer without providing any synchronization mechanism for accessing that pointer from different threads. For example, types like `Rc` and `Cell` do not implement `Sync`.

Consider an example demonstrating that `String` implements `Sync`.

```rust
fn main() {
    static S: String = String::new();
    let r1 = &S;
    let r2 = &S;

    let t1 = std::thread::spawn(move || {
        println!("{}", r1);
    });
    let t2 = std::thread::spawn(move || {
        println!("{}", r2);
    });
    let _ = t1.join();
    let _ = t2.join();
}
```

Since `String` is `Sync`, we can use references to the same object from different threads.

In this example, we also had to declare the string as a static variable. This is necessary because, in the current implementation, the compiler cannot guarantee that a spawned thread won't outlive the scope of the variable owning the `String` object captured by reference. By declaring the variable as `static`, we extend its lifetime until the program terminates.

Now, let's replace `String` with `Cell`, which does not implement `Sync`, and attempt to compile the program.

```rust,compile_fail
use std::cell::Cell;

fn main() {
    static C: Cell<i32> = Cell::new(5);

    let t1 = std::thread::spawn(move || {
        println!("{:?}", &C);
    });
    let t2 = std::thread::spawn(move || {
        println!("{:?}", &C);
    });
    let _ = t1.join();
    let _ = t2.join();
}
```

As expected, the compiler returns an error stating that `Cell` does not implement `Sync`:

```
error[E0277]: `Cell<i32>` cannot be shared between threads safely
 --> src/main.rs:4:15
  |
4 |     static c: Cell<i32> = Cell::new(5);
  |               ^^^^^^^^^ `Cell<i32>` cannot be shared between threads safely
  |
  = help: the trait `Sync` is not implemented for `Cell<i32>`
  = note: shared static variables must have a type that implements `Sync
```

Just like with `Send`, we can explicitly implement `Sync` for our type, though the full responsibility for the safety of working with unsynchronized fields then falls on us.

```rust
unsafe impl Sync for OurType {}
```

## Synchronization Mechanisms

Now that we are familiar with `Send` and `Sync`, we are ready to explore the mechanisms for synchronizing data access across different threads. The Rust standard library offers the following synchronization primitives:

* [Mutex](https://doc.rust-lang.org/std/sync/struct.Mutex.html) — Ensures exclusive access to a resource for only one thread at any given time.
* [RwLock](https://doc.rust-lang.org/std/sync/struct.RwLock.html) — Allows either multiple readers or a single exclusive writer.
* [Condvar](https://doc.rust-lang.org/std/sync/struct.Condvar.html) — Allows a thread to sleep and wait until another thread wakes it up.
* [Barrier](https://doc.rust-lang.org/std/sync/struct.Barrier.html) — Enables multiple threads to synchronize with each other at a specific point.

### Mutex

Just like in other programming languages, a mutex in Rust is a mechanism that allows only one thread to gain exclusive access to a resource at a time. All other threads wishing to access the resource at that moment are placed in a waiting state until the thread that acquired the resource releases it.

A mutex is represented by the type [`std::sync::Mutex`](https://doc.rust-lang.org/std/sync/struct.Mutex.html). To wrap an object in a mutex, use the `new` constructor method:

```rust
use std::sync::Mutex;

fn main() {
    let m: Mutex<i32> = Mutex::new(5);
}
```

To access the object inside the mutex, the `lock()` method is used. This method returns a [MutexGuard](https://doc.rust-lang.org/std/sync/struct.MutexGuard.html) smart pointer, which provides a mutable reference to the object inside the mutex (`MutexGuard` implements the `DerefMut` trait).

```rust
use std::sync::{Mutex, MutexGuard, PoisonError};

fn main() {
    let m: Mutex<i32> = Mutex::new(5);
    {
        // acquire the mutex resource
        let lock_attempt: Result<MutexGuard<'_, i32>, PoisonError<_>> = m.lock();
        let mut guard = lock_attempt.unwrap();
        *guard = 6;
    } // release the mutex
}
```

We deliberately called `lock` inside a separate block of curly braces — a scope. This is because as long as the `MutexGuard` object exists, the mutex remains locked; as soon as it is dropped (destroyed), the mutex is released. This is why it is recommended to minimize the scope in which the `MutexGuard` object exists.

Now, let's talk about working with a mutex across different threads. As we know, a thread executes a function or a closure. In either case, for a thread to work with an object, it must take ownership of it. The problem is that we cannot give ownership of a single mutex object to two different threads simultaneously. We also cannot clone a mutex. This is where the [Arc](https://doc.rust-lang.org/std/sync/struct.Arc.html) (Atomic Reference Counting) smart pointer comes in — the thread-safe version of `Rc`. We simply wrap the mutex in an `Arc`, which allows us to pass a shared reference to the mutex into different threads.

```rust
use std::{sync::{Arc, Mutex}, thread};

fn main() {
    let m_original: Arc<Mutex<i32>> = Arc::new(Mutex::new(5));

    let m_clone = m_original.clone();
    let t = thread::spawn(move|| {
        if let Ok(mut guard) = m_clone.lock() {
            *guard = 6;
        }
    });
    let _ = t.join();

    println!("{m_original:?}"); // Mutex { data: 6, poisoned: false, .. }
}
```

As you can see, the mutex was printed as `Mutex { data: 6, poisoned: false, .. }`.
Here, the `data` field is the value inside the mutex, and the poisoned field indicates whether the mutex is poisoned (we will discuss this in the next section).

---

Consider an example of a counter that is incremented simultaneously from two different threads.

```rust
use std::{sync::{Arc, Mutex}, thread::{self, JoinHandle}};

fn start_counter_thread(counter: Arc<Mutex<i32>>) -> JoinHandle<()> {
    thread::spawn(move || {
        for _ in 0 .. 1000 {
            if let Ok(mut guard) = counter.lock() {
                *guard += 1;
            }
        }
    })
}

fn main() {
    let counter = Arc::new(Mutex::new(0));

    let t1 = start_counter_thread(counter.clone());
    let t2 = start_counter_thread(counter.clone());

    let _ = t1.join();
    let _ = t2.join();

    println!("{counter:?}"); // Mutex { data: 2000, poisoned: false, .. }
}
```

Thanks to the mutex, all increment operations were performed correctly.

> [!WARNING]
> Regarding the lifetime of a mutex guard.
> 
> In cases where you only need to perform a single action with the value wrapped in a mutex, you can use a single `lock()` expression without opening a new scope. For example, if we want to extract an element from a vector wrapped in a mutex, we can write the following:
> 
> 
> ```rust
> let list: Mutex<Vec<i32>> = ...;
> let last: Option<i32> = list.lock().unwrap().pop();
> if let Some(element) = last {
>     process_element(element);
> }
> ```
> 
> On the second line, the `.lock()` call creates a guard object that is immediately destroyed upon the completion of the expression. Thus, the mutex lock duration is limited to just that one expression.
> 
> You might be tempted to shorten this code like this:
> 
> 
> ```rust
> let list: Mutex<Vec<i32>> = ...;
> if let Some(element) = list.lock().unwrap().pop() {
>     process_element(element);
> }
> ```
>
> However, if you expected the mutex to be used only to retrieve the element and then immediately release it, that is not what happens. Objects created in the header of `if-let` and `while-let` expressions live until the very end of the scope of those expressions. This means that throughout the entire execution of the `process_element` function, the mutex will remain locked.


#### Poisoned Mutex

If a thread that has already acquired a mutex terminates with a panic, the mutex is marked as **poisoned**.\
If another thread attempts to acquire a poisoned mutex, the `.lock()` method call will return a `PoisonError`.

```rust
use std::{sync::{Arc, Mutex}, thread};

fn main() {
    let m = Arc::new(Mutex::new(5));

    let m1 = m.clone();
    let t1 = thread::spawn(move|| {
        let guard = m1.lock().unwrap();
        // Trigger a panic to poison the mutex
        panic!("poisoning mutex...");
    });
    let _ = t1.join();

    // Check if the mutex is poisoned
    println!("{}", m.is_poisoned()); // true

    let lock_attempt_1 = m.lock();
    // A poisoned mutex returns PoisonError instead of Ok(Guard)
    println!("{lock_attempt_1:?}"); // Err(PoisonError { .. })

    // Nevertheless, you can still extract the Guard object from a PoisonError.
    // By extracting the Guard from PoisonError, we explicitly acknowledge that the mutex is poisoned.
    if let Err(e) = lock_attempt_1 {
        let guard = e.into_inner();
        println!("Value: {}", *guard); // 5
    }

    // A poisoned mutex can be restored to a normal state.
    m.clear_poison();
    println!("{}", m.is_poisoned()); // false
}
```

The primary purpose of `PoisonError` is to explicitly inform you that another thread that held the mutex panicked. In some scenarios, this information can help you correctly terminate or roll back an operation that the panicked thread failed to complete.

### RwLock

While a mutex provides only exclusive access to a resource, [**RwLock**](https://doc.rust-lang.org/std/sync/struct.RwLock.html) (Reader-Writer Lock) distinguishes between read access and write access. Any number of threads can simultaneously hold a read lock, but a write lock is exclusive — just like a standard mutex.
```rust
use std::sync::RwLock;

fn main() {
    let rw_lock = RwLock::new(5);

    { // Read acquisition
        let lock_attempt = rw_lock.read();
        if let Ok(guard) = lock_attempt {
            println!("Read: {}", *guard);
        }
    }

    { // Write acquisition
        let lock_attempt = rw_lock.write();
        if let Ok(mut guard) = lock_attempt {
            *guard = 10;
            println!("Updated: {}", *guard);
        }
    }

    println!("{rw_lock:?}");
}
```

Just like a mutex, an `RwLock` must be wrapped in an `Arc` to be shared across multiple threads.

An `RwLock` becomes "poisoned" only if a thread that has acquired a write lock panics. A panic during a read lock does not lead to poisoning.

### Condvar

The [Condvar](https://doc.rust-lang.org/std/sync/struct.Condvar.html) (Condition Variable) type allows one thread to wait until another thread changes a specific variable to an expected value.

A `Condvar` is always used in tandem with a mutex, and this interaction is easiest to understand through an example.

Let's take a classic task for `Condvar`: one thread needs to wait for another thread to complete a specific action. This is implemented as follows:

* One thread waits until a boolean variable, wrapped in a mutex, changes its value from `false` to `true`.
* The other thread performs some work and then sets that boolean variable to `true`.

```rust
use std::sync::{Arc, Mutex, Condvar};
use std::thread;

fn main () {
    // Create a Condvar with a boolean flag
    let cond = Arc::new((Mutex::new(false), Condvar::new()));

    let cond_copy = Arc::clone(&cond);
    thread::spawn(move || {
        // This thread simulates some preparation,
        // after which the condvar flag will be set to true
        let (mutex, cvar) = &*cond_copy;
        let mut flag_guard = mutex.lock().unwrap();
        *flag_guard = true;
        cvar.notify_one();
    });

    let (mutex, cvar) = &*cond;
    let mut flag_guard = mutex.lock().unwrap();
    // Here we wait until the spawned thread sets the flag to true
    while !(*flag_guard) {
        flag_guard = cvar.wait(flag_guard).unwrap();
    }
}
```

In the line `flag_guard = cvar.wait(flag_guard).unwrap();`, the main thread "goes to sleep" during the `wait` call and stays there until `notify_one` is called on the same `Condvar` object. Once `notify_one` is called in the second thread, the main thread wakes up and checks the updated flag value hidden behind the `MutexGuard`.

Note that passing the `MutexGuard` into the `wait` call releases the mutex. Without this, the second thread would be unable to lock the mutex in the line `let mut flag_guard = mutex.lock().unwrap()`, resulting in a deadlock.

In addition to `notify_one`, `Condvar` also has a `notify_all` method. This is used if multiple threads are calling `wait` on the same `Condvar` object; `notify_one` wakes up only one waiting thread, whereas `notify_all` wakes them all up.

### Barrier

A [Barrier](https://doc.rust-lang.org/std/sync/struct.Barrier.html) is a mechanism that allows a set of threads to synchronize with each other, making them wait until all of them are ready to proceed.

```rust
use std::sync::{Arc, Barrier, Mutex};
use std::thread;

const WORKERS_NUM: usize = 10;

fn main() {
    let data = (0..100).collect::<Vec<_>>();
    let mutex = Arc::new(Mutex::new(data));
    let barrier = Arc::new(Barrier::new(WORKERS_NUM));

    let mut workers = Vec::new();
    for _ in 0 .. WORKERS_NUM {
        let mutex_clone = mutex.clone();
        let barrier_clone = barrier.clone();
        let t = thread::spawn(move || {
            loop {
                // Wait here until all 10 threads reach this line
                barrier_clone.wait();
                let Some(element) = mutex_clone.lock().unwrap().pop() else {
                    break;
                };
                println!("Processing {element} by {:?}", thread::current().id());
            }
        });
        workers.push(t);
    }
    workers.into_iter().for_each(|t| t.join().unwrap());
}
```

By running this code, you can see that the data array (a vector of 100 elements) is processed in batches of 10, with each element being handled by an individual thread.

## scoped thread

One of the more pesky limitations of standard threads is that even if a thread is spawned within a function and finishes its work before that function returns (because you called `.join()`), it still cannot borrow local variables by reference without taking full ownership.

For example:

```rust
fn main() {
    let s = "Hello".to_string();
    let thread = std::thread::spawn(|| {
        println!("{}", &s);
    });
    let _ = thread.join(); // The thread finishes here
}
```

Compiling this program results in the following error:

```
closure may outlive the current function, but it borrows `s`,
which is owned by the current function
```

The spawned thread is guaranteed to terminate before the end of the `main` function's scope. However, the Rust compiler isn't "smart" enough to analyze `.join()` calls to verify this. It assumes the thread might outlive the function, making it unsafe to hold a reference to `s`.

For exactly these situations (where a thread's lifetime is strictly bound to the scope in which it was created) we have [scoped thread](https://doc.rust-lang.org/std/thread/fn.scope.html).

Let's look at an example:

```rust
fn main() {
    let s = "Hello".to_string();

    // Create a thread scope
    std::thread::scope(|scope| {

        // Spawn a thread within the scope
        scope.spawn(|| {
            println!("{}", &s);
        });
    }); // All threads in the scope are guaranteed to be joined here
}
```

Scoped threads offer two major benefits:

* They can access local variables of the parent function directly by reference, without needing `move` or `Arc`.
* They are guaranteed to terminate by the end of the `scope` block. It’s as if the compiler automatically calls `.join()` for every thread spawned within that block before allowing execution to proceed.

Let's rewrite our previous barrier example using scoped threads to see how much cleaner the code becomes.

```rust
use std::sync::{Barrier, Mutex};
use std::thread;

const WORKERS_NUM: usize = 10;

fn main() {
    let data = (0..100).collect::<Vec<_>>();
    let mutex = Mutex::new(data);
    let barrier = Barrier::new(WORKERS_NUM);

    thread::scope(|s| {
        // Scope for launching threads
        for _ in 0..WORKERS_NUM {
            s.spawn(|| {
                loop {
                    barrier.wait();
                    let Some(element) = mutex.lock().unwrap().pop() else {
                        break;
                    };
                    println!("Processing {element} by {:?}", thread::current().id());
                }
            });
        }
    });
}
```

As you can see, the code is significantly more concise. We've completely eliminated the need for `Arc` wrappers and the boilerplate logic for collecting and joining thread handles.

## Atomics

Rust provides atomic wrappers for primitive data types that can be safely used in multi-threaded environments.

| Primitive Type |        Atomic Equivalent          |
| -------------- | --------------------------------- |
| `bool`         | `std::sync::atomic::AtomicBool`   |
| `u8`           | `std::sync::atomic::AtomicU8`     |
| `u16`          | `std::sync::atomic::AtomicU16`    |
| `u32`          | `std::sync::atomic::AtomicU32`    |
| `u64`          | `std::sync::atomic::AtomicU64`    |
| `i8`           | `std::sync::atomic::AtomicI8`     |
| `i16`          | `std::sync::atomic::AtomicI16`    |
| `i32`          | `std::sync::atomic::AtomicI32`    |
| `i64`          | `std::sync::atomic::AtomicI64`    |
| `usize`        | `std::sync::atomic::AtomicUsize`  |
| `isize`        | `std::sync::atomic::AtomicIsize`  |
| `*mut T`       | `std::sync::atomic::AtomicPtr<T>` |

All numeric atomic types allow you to atomically read and write values, as well as perform atomic arithmetic and logical operations.

For example, let's create a counter based on [AtomicI32](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicI32.html) and increment it from different threads.

```rust
use std::{
    sync::atomic::{AtomicI32, Ordering},
    thread,
};

fn main() {
    let a = AtomicI32::new(0); // initialize the atomic with zero
    thread::scope(|s| {
        for _ in 0..1000 {
            s.spawn(|| {
                for _ in 0..1000 {
                    a.fetch_add(1, Ordering::Relaxed); // increment
                }
            });
        }
    });
    println!("{}", a.load(Ordering::Relaxed)); // 1000000
}
```

After performing 1,000 increments from 1,000 threads, we will expectedly get a counter value of 1,000,000.

> [!TIP]
> <details>
> <summary>For comparison, here is the same counter using a raw pointer instead of an atomic</summary>
> 
> ```rust
> use std::thread;
> 
> fn main() {
>     let mut a = 0;
>     let address = (&raw mut a).addr();
>     thread::scope(|s| {
>         for _ in 0..1000 {
>             s.spawn(|| {
>                 for _ in 0..1000 {
>                     unsafe {
>                         *(address as *mut i32) += 1;
>                     }
>                 }
>             });
>         }
>     });
>     println!("{}", a); // 862886
> }
> ```
> </details>


The [fetch_add](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicI32.html#method.fetch_add) method, which performs addition, takes not only the value to add but also an [Ordering](https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html) parameter. This argument controls the so-called **memory ordering**, which defines the guarantees we get about the order in which atomic operations are observed across threads.

The reason this matters is that the compiler may reorder instructions so that their execution order does not match the order in the source code. This is done to optimize performance and does not affect the program’s logic. If the compiler sees that the value of variable `y` depends on the value of variable `x`, it will ensure that `x` is computed before `y`. However, the compiler can track such dependencies only within a single thread. When atomics are involved, it cannot determine the ordering of interactions between different atomic variables across different threads. The `Ordering` parameter exists to explicitly define these guarantees. It can take the following values:

* `Relaxed` — still guarantees consistency for a single atomic variable when accessed from multiple threads, but makes <ins>no guarantees</ins> about the relative ordering of operations between different atomic variables. This means that two threads using `Relaxed` ordering may observe operations on two different atomic variables in different orders. For example, if the first thread writes to atomic `a` and then to atomic `b`, another thread may observe these writes in the opposite order.

> [!NOTE]
> In most backend applications, atomics are typically used only as counters for statistics, and almost always with `Relaxed` ordering.

* `Release` — used for write operations. When an atomic write is performed with `Release` ordering, all preceding operations (including non-atomic operations and `Relaxed` atomic writes) must become visible to other threads that read this atomic using `Acquire` ordering.\
  For example, if we write to atomic variable `a` and then write to atomic `b` with `Release` ordering, another thread that reads `b` with `Acquire` ordering is guaranteed to see the changes to `a` that happened before the write to `b`.
* `Acquire` — used for read operations from an atomic variable whose value was written with `Release` ordering.
* `AcqRel` — used for operations that have both read and write semantics, such as `fetch_add` (it first reads the value, then computes the addition, and finally writes the result using a _compare-and-swap_ operation). When `fetch_add` is performed with `AcqRel` ordering, the read happens with `Acquire` semantics and the write with `Release`.
* `SeqCst` — used for both reads and writes. This provides the strongest synchronization guarantees, enforcing a single global order of atomic operations. Use this ordering when your logic depends on the exact sequence of atomic changes. However, keep in mind that it has the lowest performance among all ordering modes.

***

In addition to `fetch_add`, integer atomic also provide following methods:
* substraction — `fetch_sub`
* bitwise OR — `fetch_or`
* bitwise AND — `fetch_and`
* bitwise NOT AND — `fetch_nand`
* exclusive OR — `fetch_xor`

***

Atomics also support the Compare-And-Exchange operation, which is used when you need to read the current atomic value, compute a new value based on it, and then write the result back. A simple combination of the `load` and `store` methods is not suitable in this scenario, because between the `load` and `store` calls (while the new value is being computed), another thread may write to the same atomic. As a result, the current thread would compute a new value based on stale data.

The compare-and-exchange operation (in fact, the same as compare-and-swap) is provided by the following method:

```rust
pub fn compare_exchange(
    &self,
    expected: тип_значения_атомика,
    new: тип_значения_атомика,
    success_ordering: Ordering,
    failure_ordering: Ordering,
) -> Result<тип, тип>
```

This method takes the expected current value of the atomic — `expected`, and the new value — `new`. The new value is written to the atomic only if the expected value matches the actual current value stored in the atomic.

The method returns `Ok(actual_previous_value)` if the write was performed, or `Err(actual_current_value)` if the write was not performed.

Memory ordering parameters:

* `success_ordering` specifies the memory ordering for the successful read-modify-write operation that occurs when the expected value matches the actual current value.
* `failure_ordering` specifies the memory ordering for the load operation used to read the actual current value if it does not match the expected value.

An example of using `compare_exchange`: let’s rewrite our counter increment from a thousand threads using `compare_exchange`.

```rust
use std::{
    sync::atomic::{AtomicI32, Ordering},
    thread,
};

fn main() {
    let a = AtomicI32::new(0);
    thread::scope(|s| {
        for _ in 0..1000 {
            s.spawn(|| {
                for _ in 0..1000 {
                    let mut old_val = a.load(Ordering::Relaxed);
                    loop {
                        let new_val = old_val + 1;
                        let r = a.compare_exchange(
                            old_val,
                            new_val,
                            Ordering::Relaxed,
                            Ordering::Relaxed,
                        );
                        if let Err(actual_val) = r {
                            old_val = actual_val;
                        } else {
                            break;
                        }
                    }
                }
            });
        }
    });
    println!("{:?}", a); // 1000000
}
```

Just for an experiment you can try to replace the inner loop with follwoing:

```rust
for _ in 0..1000 {
    let old_val = a.load(Ordering::Relaxed);
    let new_val = old_val + 1;
    a.store(new_val, Ordering::Relaxed);
}
```

and check the result.

## Thread local storage

Thread-local storage is a mechanism that provides data storage local to a specific thread.

The idea is that in code we work with a thread-local variable as if it were global. However, for each thread, this “global” variable has its own independent value.

A thread-local variable is declared using the [thread_local](https://doc.rust-lang.org/std/macro.thread_local.html) macro:

```rust
thread_local! {
    static VARIABLE: Type = value;
}
```

After that, you can work with the thread-local variable as if it were a regular global variable. Any changes to the variable will be visible only within the same thread.

Let’s consider a simple example that clearly demonstrates that each thread has its own value of a thread-local variable.

```rust
use std::{cell::Cell, thread};

thread_local! {
    pub static NUM: Cell<u32> = const { Cell::new(0) };
}

fn print_num() {
    println!("{}", NUM.get());
}

fn main() {
    let t1 = thread::spawn(|| {
        NUM.set(1);
        print_num();
    });
    let t2 = thread::spawn(|| {
        NUM.set(2);
        print_num();
    });
    let _ = t1.join();
    let _ = t2.join();
}
```

The program prints:

```
1
2
```

Thread-local variables are often used in web servers. For example, at the very beginning of request processing, we place information about the user session into a thread-local variable. This information then becomes available throughout the entire request-handling call chain. Without thread-local storage, we would have to pass the session object through every function call along the way.

## Channels { #channels }

For communication between threads, the Rust standard library provides channels.

Essentially, a channel is a synchronized queue: one thread adds items to the end of the queue, while another thread extracts items from the beginning.

The standard library provides the [mpsc](https://doc.rust-lang.org/std/sync/mpsc/index.html) (multiple producers, single consumer) channel, which allows multiple threads to send messages into the channel, but only one thread to read them.

> [!NOTE]
> If you need multiple threads to be able to read from a channel, the [crossbeam-channel](https://crates.io/crates/crossbeam-channel) crate provides an mpmc (multiple producers, multiple consumers) channel.
> 
> Additionally, the Rust standard library has its own [mpmc](https://doc.rust-lang.org/std/sync/mpmc/index.html) channel; however, as of now (Rust 1.92), it is only available in the Nightly build.

A channel is created using one of two functions:

* [channel](https://doc.rust-lang.org/std/sync/mpsc/fn.channel.html) — creates an unbounded channel.
* [sync_channel](https://doc.rust-lang.org/std/sync/mpsc/fn.sync_channel.html) — creates a bounded channel of a fixed size. If the channel is full, any attempt to send a message will block the sender thread until the receiver extracts a message, freeing up space.

Both `channel` and `sync_channel` return a tuple containing two elements: [Sender](https://doc.rust-lang.org/std/sync/mpsc/struct.Sender.html) and [Receiver](https://doc.rust-lang.org/std/sync/mpsc/struct.Receiver.html).

```rust
pub fn channel<T>() -> (Sender<T>, Receiver<T>)
pub fn sync_channel<T>(bound: usize) -> (SyncSender<T>, Receiver<T>)
```

The `Sender` object is used to push messages into the channel, while the `Receiver` is used to read them.

Let’s look at a simple example: one thread sends numbers into a channel, and another thread retrieves them and prints them to the console.

```rust
use std::{sync::mpsc, thread};

// A wrapper for the data we will pass through the channel
enum Element {
    Num(i32), // the next number
    Finish,   // completion flag
}

fn main() {
    let (producer, receiver) = mpsc::channel::<Element>();
    let t1 = thread::spawn(move || {
        for i in 0..5 {
            let _ = producer.send(Element::Num(i));
        }
    });
    let t2 = thread::spawn(move || {
        while let Ok(msg) = receiver.recv() {
            match msg {
                Element::Num(i) => println!("{i}"),
                Element::Finish => break,
            }
        }
    });
    let _ = t1.join();
    let _ = t2.join();
}
```

In the previous example, we used an unbounded channel. Now let’s see how a fixed-size (bounded) channel behaves.

We will create a channel with a capacity of 3 messages. One thread will send messages and print how long the operation took, while the other thread will wait 1 second in a loop before extracting each message.

```rust
use std::{sync::mpsc, thread, time::{Duration, Instant}};

fn main() {
    let (snd, rcv) = mpsc::sync_channel::<i32>(3);
    let t1 = thread::spawn(move || {
        for i in 0..5 {
            let start = Instant::now();
            let _ = snd.send(i);
            println!("Took {} millis to send msg", start.elapsed().as_millis());
        }
    });
    let _ = thread::spawn(move || {
        loop {
            thread::sleep(Duration::from_secs(1));
            match rcv.recv() {
                Ok(_) => (),
                Err(_) => break,
            }
        }
    });
    let _ = t1.join();
}
```

The program prints:

```
Took 0 millis  to send msg
Took 0 millis  to send msg
Took 0 millis  to send msg
Took 1000 millis  to send msg
Took 1000 millis  to send msg
```

Since the channel capacity is 3, the first three messages are sent instantly. After that, the sender thread is blocked and must wait for space to become available. Because the receiver pauses for 1 second between reads, the sender is forced to wait 1 second for each subsequent message delivery.

## What to Read

In this chapter, we provided a brief overview of multithreading in Rust. This material will most likely be sufficient for building backend applications. However, if you’d like to dive deeper into the topic of multithreading, be sure to check out the free book Rust Atomics and Locks by Mara Bos.

[https://marabos.nl/atomics/](https://marabos.nl/atomics/)

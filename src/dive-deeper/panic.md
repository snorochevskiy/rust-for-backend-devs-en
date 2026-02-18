# Panic

When an invalid action occurs in a program (such as division by zero or calling `unwrap()` on a `None` value) a panic occurs.

For example:

```rust,should_panic
fn main() {
    let a: Option<i32> = None;
    a.unwrap();
}
```

This will output:

```
$ cargo run
thread 'main' panicked at src/main.rs:3:7:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

A panic crashes the program and prints information about the error to the standard output.

You can also trigger a panic manually using the [panic](https://doc.rust-lang.org/std/macro.panic.html) macro. Its syntax is the same as the `println!` macro, but instead of printing to the console and continuing, it terminates the program and prints the provided message.

For example:

```rust,should_panic
fn main() {
    let a = 5;
    panic!("Ending the program here. Num: {}", a);
}
```

Output:

```
$ cargo run
thread 'main' panicked at src/main.rs:3:5:
Ending the program here. Num: 5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

## Catching a Panic

By default, a panic terminates the program's execution. However, this isn't always the desired behavior. For instance, when developing a web server, if a panic occurs within a single request handler, we usually prefer to return an HTTP 500 error rather than crashing the entire server.

The standard library provides a wrapper function called [catch_unwind](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html), which takes a closure and executes it.

```rust,noplayground
pub fn catch_unwind<F: FnOnce() -> R + UnwindSafe, R>(f: F) -> Result<R>
```

If the closure completes successfully, its result is returned wrapped in `Ok`. If a panic occurs during execution, it returns an `Err`.

> [!NOTE]
> Recall that if a function expects an `FnOnce` closure as an argument, you can pass `FnMut`, `Fn`, or a simple function pointer `fn`.

Consider this example:

```rust
use std::panic;
use std::panic::catch_unwind;
use std::any::Any;

fn main() {
    let result: Result<(), Box<dyn Any + Send + 'static>> = catch_unwind(|| {
        panic!("My panic msg");
    });
    println!("Result: '{:?}'", result);
    println!("Continue working...");
}
```

This program prints:

```
$ cargo run
thread 'main' panicked at src/main.rs:5:9:
My panic msg
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
Result: 'Err(Any { .. })'
Continue working...
```

The fact that `catch_unwind` returns the panic as an `Err` containing `Any` suggests that a panic can carry an object. This is true. For cases where you need to pass an object through a panic, the [panic_any](https://doc.rust-lang.org/std/panic/fn.panic_any.html) function exists, which accepts any type that implements `Any`.

```rust
pub fn panic_any<M: 'static + Any + Send>(msg: M) -> !
```

Let's write a simple function that simulates a user request handler. This function takes a text request and returns a text response. In case of an error, the function can throw a panic containing an object describing the problem.

```rust
use std::{any::Any, panic, panic::panic_any};

/// Error type to be passed through panic
#[derive(Debug)]
enum ProcessingErr {
    NotAuthorized,
    AnotherImportantError,
}

/// A request handler that might panic
fn serve_request(req: &str) -> String {
    panic_any(ProcessingErr::NotAuthorized);
}

fn main() {
    let request = "Some request"; // Simulates a request

    let closure_result: Result<String, Box<dyn Any + Send + 'static>> =
        panic::catch_unwind(|| serve_request(request));

    if let Err(a) = closure_result {
        if let Some(panic_obj) = a.downcast_ref::<ProcessingErr>() {
            println!("Panic object: {panic_obj:?}");
        }
    }
}
```

The program will print:

```
# cargo run
thread 'main' panicked at src/main.rs:12:5:
Box<dyn Any>
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
Panic object: NotAuthorized
```

Using this method, we can retrieve information about the issue that caused the panic.

> [!CAUTION]
> Keep in mind that this program was written solely to demonstrate the ability to pass objects through panics. Using panics as a substitute for exceptions for regular error handling is a very bad idea. Panics should only be used for serious problems that are not part of normal operation. For everything else, use `Result`.

## Panic Hook

As we noted in the previous example, even though we catch a panic using the `catch_unwind` function, a panic message and the location where it was triggered are still printed to the console:

```
thread 'main' panicked at src/main.rs:5:9:
My panic msg
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

This happens because, by default, a standard panic hook is set, which executes before the panic is caught by `catch_unwind`.

If we need different behavior, we can define our own panic hook using the [set_hook](https://doc.rust-lang.org/std/panic/fn.set_hook.html) function.

```rust
use std::panic::{self, PanicHookInfo};

fn my_panic_hook<'a>(p: &PanicHookInfo<'a>) {
    println!("Hello from panic hook");
    println!("Location: {:?}", p.location());
    println!("Payload: {:?}", p.payload());
}

fn main() {
    panic::set_hook(Box::new(my_panic_hook));

    panic!("Original panic msg");
}
```

This program will output:

```
$cargo run
Hello from panic hook
Location: Some(Location { file: "src/main.rs", line: 12, column: 5 })
Payload: Any { .. }
```

## Panic-Inducing Macros

To wrap up the topic of macros, it is worth mentioning a few others that also trigger a panic.

### todo and unimplemented

The [todo](https://doc.rust-lang.org/std/macro.todo.html) and [unimplemented](https://doc.rust-lang.org/std/macro.unimplemented.html) macros serve the same purpose: they act as placeholders for code that hasn't been implemented yet. A distinctive feature of these macros is that they "return" whatever type is expected of them. For example:

```rust
fn my_func() -> String {
    todo!("Don't forget to implement")
}

fn main() {
    let s = my_func();
    println!("{s}");
}
```

This code compiles without errors, but the program panics when you try to execute it:

```
$ cargo run
thread 'main' panicked at src/main.rs:2:5:
not yet implemented: Don't forget to implement
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

The `my_func` function is supposed to return a `String`, so the call to the `todo` macro effectively "returns" the `String` type. This is very convenient because you can insert `todo` anywhere, no matter how complex the expected type is, and the compiler will be satisfied.

Furthermore, `todo` can be used in any context, not just when returning values from a function:

```rust
let r: Result<(), Box<dyn Error>> = todo!();
```

The `unimplemented` macro works exactly like `todo`. The difference is purely semantic: `todo` implies that the functionality should be implemented in the current development iteration, while `unimplemented` suggests that the functionality might be implemented later.

### unreachable

The [unreachable](https://doc.rust-lang.org/std/macro.unreachable.html) macro also triggers a panic. However, unlike `todo` and `unimplemented`, it is used for sections of code that technically could be executed, but in practice, execution should never reach them (usually due to prior argument validation).

For example:

```rust
/// This function expects API version 1 or 2.
/// API version validation is the responsibility of the calling code.
fn get_user_by_id(id: u64, api_version: u32) {
    match api_version {
        1 => v1_get_user_by_id(id),
        2 => v2_get_user_by_id(id),
        _ => unreachable!("Unknown api version"),
    }
}
```

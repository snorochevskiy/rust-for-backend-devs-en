# Error Handling

In the chapter about the [Result](../rust-basics/result.md) type, we covered the basics of error handling. In this chapter, we will explore how errors are typically handled in backend applications.

## thiserror

Let’s imagine we are developing a service responsible for reserving products. To start, the service will contain only one function that takes two parameters: the product ID to reserve and the desired quantity.

It is obvious that at least two errors can occur in such functionality:

* Attempting to reserve an unknown product.
* Attempting to reserve more items than are currently in stock.

We can write the code for this service as follows:

```rust,noplayground
/// A type representing a successfully created reservation object
struct Reservation {
    reservation_id: u64,
    product_id: u64,
    quantity: u64,
}

trait ReservationService {
    /// Reserves the specified product in the specified quantity
    /// * product ID
    /// * number of items
    fn reserve(&self, id: u64, quantity: u64) -> Result<Reservation, ReserveError>;
}

enum ReserveError {
    NoSuchProduct { id: u64 },
    NotEnough { asked: u64, available: u64 },
}
```

We also know from the section [about the Error trait](../rust-basics/result.md#trait-error) that it is recommended to implement the [std::error::Error](https://doc.rust-lang.org/std/error/trait.Error.html) trait for types representing errors. So, let's implement it for our error type — `ReserveError`:

```rust,noplayground
#[derive(Debug)]
enum ReserveError {
  NoSuchProduct { id: u64 },
  NotEnough { asked: u64, available: u64 },
}

impl std::fmt::Display for ReserveError {
  fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
    use ReserveError::*;
    match self {
      NoSuchProduct { id } =>
        write!(f, "No product with ID {id}"),
      NotEnough {asked, available} =>
        write!(f, "Asked {asked}, but available {available}"),
    }
  }
}

impl std::error::Error for ReserveError { }
```

It’s easy to see that implementing the Error trait is cumbersome and involves boilerplate code that would be identical in Error implementations for other types. Fortunately, there is a third-party library called [thiserror](https://crates.io/crates/thiserror) that greatly simplifies the creation of error types.

Here is what the equivalent definition of our `ReserveError` type looks like using thiserror:

```rust,noplayground
use thiserror::Error;

#[derive(Debug, Error)]
enum ReserveError {
  #[error("No product with ID {id}")]
  NoSuchProduct { id: u64 },
  #[error("Asked {asked}, but available {available}")]
  NoEnoughQuantity { asked: u64, available: u64 },
}
```

The thiserror library contains a procedural macro that is triggered for enums and structs annotated with `#[derive(thiserror::Error)]`. This macro generates the `std::fmt::Display` and `std::error::Error` implementations, effectively doing exactly what we previously did manually.

To better understand how this works as a whole, let's write a small program using our product reservation functionality.

Create a new project:

```
cargo new test_rust
```

Add thiserror to `Cargo.toml`:

```toml
[package]
name = "test_rust"
version = "0.1.0"
edition = "2024"

[dependencies]
thiserror = "1"
```

Now for `src/main.rs`. Let’s write a storage implementation from which we will reserve products. For simplicity, we will store the products in a hash map.

```rust
use std::{
    collections::HashMap,
    sync::{Mutex, atomic::{AtomicU64, Ordering}},
};

#[derive(Debug)]
struct Reservation {
    reservation_id: u64,
    product_id: u64,
    quantity: u64,
}

trait ReservationService {
    fn reserve(&self, id: u64, quantity: u64) -> Result<Reservation, ReserveError>;
}

#[derive(Debug, thiserror::Error)]
enum ReserveError {
    #[error("No product with ID {id}")]
    NoSuchProduct { id: u64 },
    #[error("Asked {asked}, but available {available}")]
    NotEnough { asked: u64, available: u64 },
}

// Primitive warehouse implementation as a hash map: product ID -> quantity
struct ReservationImpl {
    storage: Mutex<HashMap<u64, u64>>,
    last_id: AtomicU64,
}

impl ReservationService for ReservationImpl {
    fn reserve(&self, id: u64, quantity: u64) -> Result<Reservation, ReserveError> {
        let mut guard = self.storage.lock().unwrap();
        if let Some(stock) = guard.get_mut(&id) {
            if *stock < quantity {
                Err(ReserveError::NotEnough {
                    asked: quantity,
                    available: *stock,
                })
            } else {
                *stock -= quantity;
                Ok(Reservation {
                    reservation_id: self.last_id.fetch_add(1, Ordering::Relaxed),
                    product_id: id,
                    quantity,
                })
            }
        } else {
            Err(ReserveError::NoSuchProduct { id })
        }
    }
}

fn main() {
    let mut products = HashMap::new();
    products.insert(111, 50); // Product 111 with a quantity of 50 items

    let reservation_service = ReservationImpl {
        storage: Mutex::new(products),
        last_id: AtomicU64::new(0),
    };

    println!("{:?}", reservation_service.reserve(112, 1));
    // Err(NoSuchProduct { id: 112 })

    println!("{:?}", reservation_service.reserve(111, 51));
    // Err(NoEnoughQuantity { asked: 51, available: 50 })

    println!("{:?}", reservation_service.reserve(111, 10));
    // Ok(Reservation { reservation_id: 1, product_id: 111, quantity: 10 })
}
```

## Wrapping Errors

Now, let's expand our application: we will add functionality for scheduling home delivery and a purchase feature that combines both reservation and shipping.

The logic is as follows:

![](img/error_handling-wrapping.svg)

The API will be represented by three services, each consisting of:
* A trait describing the service interface.
* A struct representing the result of a successful service call.
* An error type representing the problem in case of a failed call.

Reservation:

```rust,noplayground
struct Reservation {
    reservation_id: u64,
    product_id: u64,
    quantity: u64,
}

#[derive(Debug, thiserror::Error)]
enum ReserveError {
    #[error("No product with ID {id}")]
    NoSuchProduct { id: u64 },
    #[error("Asked {asked}, but available {available}")]
    NotEnough { asked: u64, available: u64 },
}

trait ReservationService {
    fn reserve(
        &self, id: u64, quantity: u64
    ) -> Result<Reservation, ReserveError>;
}
```

Shipping:

```rust,noplayground
struct Shipment {
    shipment_id: u64,
    address: String,
    reservation_id: u64,
}

#[derive(Debug, thiserror::Error)]
enum ShipmentError {
    #[error("Invalid address: {address}")]
    InvalidAddress { address: String },
}

trait ShipmentService {
    fn schedule_shipment(
        &self, reservation: &Reservation, address: &str
    ) -> Result<Shipment, ShipmentError>;
}
```

Purchase (Reservation + Shipping):

```rust,noplayground
struct Purchase {
    purchase_id: u64,
    reservation_id: u64,
    shipment_id: u64,
}

#[derive(Debug, Error)]
enum PurchaseError {
    #[error("Nested servation error: (0)")]
    ReservationFailed(#[from] ReserveError)
    #[error("Nested shipping error: (0)")]
    ShippingFailed(#[from] ShipmentError)
}

trait PurchaseService {
    fn purchase(
        &self, id: u64, quantity: u64, addr: &str
    ) -> Result<Purchase, PurchaseError>;
}
```

Notice that PurchaseError simply wraps errors from the underlying services using the `#[from]` attribute. This attribute tells thiserror to generate the corresponding implementation of the [From](https://doc.rust-lang.org/std/convert/trait.From.html) trait, which we covered in the chapter on [Core Traits](common-traits.md#from-into).

For example, the following will be generated for the code above:

```rust,noplayground
impl From<ReservationError> for PurchaseError { ... }
impl From<ShipmentError> for PurchaseError { ... }
```

Thus, by "wrapping" one error inside another, we simultaneously preserve information about the root cause of the problem and provide errors specific to the current API.

Let’s extend our example to demonstrate how an error originating in the underlying `ReservationService` or `ShipmentService` is returned from the `PurchaseService::purchase` method wrapped in a `PurchaseError`.

Code for `src/main.rs`:

```rust,edition2024
# use thiserror;
use std::{
    collections::HashMap,
    sync::{Arc, Mutex, atomic::{AtomicU64, Ordering}},
};

// ----- Reservation functionality
#[derive(Debug)]
struct Reservation {
    id: u64,
    product_id: u64,
    quantity: u64,
}

#[derive(Debug, thiserror::Error)]
enum ReserveError {
    #[error("No product with ID {id}")]
    NoSuchProduct { id: u64 },
    #[error("Asked {asked}, but available {available}")]
    NotEnough { asked: u64, available: u64 },
}

trait ReservationService {
    fn reserve(&self, id: u64, quantity: u64) -> Result<Reservation, ReserveError>;
}

struct ReservationImpl {
    storage: Mutex<HashMap<u64, u64>>,
    last_id: AtomicU64,
}

impl ReservationImpl {
    fn new(storage: HashMap<u64, u64>) -> ReservationImpl {
        ReservationImpl {
            storage: Mutex::new(storage),
            last_id: AtomicU64::new(0),
        }
    }
}

impl ReservationService for ReservationImpl {
    fn reserve(&self, id: u64, quantity: u64) -> Result<Reservation, ReserveError> {
        let mut guard = self.storage.lock().unwrap();
        if let Some(stock) = guard.get_mut(&id) {
            if *stock < quantity {
                Err(ReserveError::NotEnough {
                    asked: quantity,
                    available: *stock,
                })
            } else {
                *stock -= quantity;
                Ok(Reservation {
                    id: self.last_id.fetch_add(1, Ordering::Relaxed),
                    product_id: id,
                    quantity,
                })
            }
        } else {
            Err(ReserveError::NoSuchProduct { id })
        }
    }
}

// ----- Shipping functionality
struct Shipment {
    id: u64,
    address: String,
    reservation_id: u64,
}

#[derive(Debug, thiserror::Error)]
enum ShipmentError {
    #[error("Invalid address: {address}")]
    InvalidAddress { address: String },
}

trait ShipmentService {
    fn schedule_shipment(
        &self,
        reservation: &Reservation,
        address: &str,
    ) -> Result<Shipment, ShipmentError>;
}

#[derive(Debug)]
struct ShipmentImpl {
    last_id: AtomicU64,
}

impl ShipmentImpl {
    fn new() -> ShipmentImpl {
        ShipmentImpl {
            last_id: AtomicU64::new(0),
        }
    }
}

impl ShipmentService for ShipmentImpl {
    fn schedule_shipment(
        &self, reservation: &Reservation, address: &str,
    ) -> Result<Shipment, ShipmentError> {
        if address.split(" ").count() < 2 {
            Err(ShipmentError::InvalidAddress { address: address.to_string() })
        } else {
            Ok(Shipment {
                id: self.last_id.fetch_add(1, Ordering::Relaxed),
                address: address.to_string(),
                reservation_id: reservation.id,
            })
        }
    }
}

// ----- Purchase functionality
#[derive(Debug)]
struct Purchase {
    id: u64,
    reservation_id: u64,
    shipment_id: u64,
}

#[derive(Debug, thiserror::Error)]
enum PurchaseError {
    #[error("Nested servation error: (0)")]
    ReservationFailed(#[from] ReserveError),
    #[error("Nested shipping error: (0)")]
    ShippingFailed(#[from] ShipmentError),
}

trait PurchaseService {
    fn purchase(
        &self, id: u64, quantity: u64, addr: &str
    ) -> Result<Purchase, PurchaseError>;
}

struct PurchaseImpl {
    reservation_service: Arc<dyn ReservationService>,
    shipment_service: Arc<dyn ShipmentService>,
    last_id: AtomicU64,
}

impl PurchaseImpl {
    fn new(
        reservation_service: Arc<dyn ReservationService>,
        shipment_service: Arc<dyn ShipmentService>,
    ) -> PurchaseImpl {
        PurchaseImpl {
            reservation_service,
            shipment_service,
            last_id: AtomicU64::new(0),
        }
    }
}

impl PurchaseService for PurchaseImpl {
    fn purchase(
        &self, id: u64, quantity: u64, addr: &str
    ) -> Result<Purchase, PurchaseError> {
        let reservation = self.reservation_service.reserve(id, quantity)?;
        let shipment = self
            .shipment_service
            .schedule_shipment(&reservation, addr)?;
        Ok(Purchase {
            id: self.last_id.fetch_add(1, Ordering::Relaxed),
            reservation_id: reservation.id,
            shipment_id: shipment.id,
        })
    }
}

// Preparing mocks for services
fn initialize_purchase_service() -> Arc<dyn PurchaseService> {
    let mut products = HashMap::new();
    products.insert(111, 50); // Product 111 in quantity of 50
    let reservation_service = ReservationImpl::new(products);

    let shipment_service = ShipmentImpl::new();

    let purchase_service =
        PurchaseImpl::new(Arc::new(reservation_service), Arc::new(shipment_service));

    Arc::new(purchase_service)
}

fn main() {
    let purchase_service = initialize_purchase_service();

    println!("{:?}", purchase_service.purchase(112, 1, "addr 1"));
    // Err(ReservationFailed(NoSuchProduct { id: 112 }))

    println!("{:?}", purchase_service.purchase(111, 51, "addr 1"));
    // Err(ReservationFailed(NotEnough { asked: 51, available: 50 }))

    println!("{:?}", purchase_service.purchase(111, 10, "invalid"));
    // Err(ShippingFailed(InvalidAddress { address: "invalid" }))

    println!("{:?}", purchase_service.purchase(111, 10, "addr 1"));
    // Ok(Purchase { id: 0, reservation_id: 1, shipment_id: 0 })
}
```



## Box\<dyn Error>

There are several situations where it’s simply impossible to handle an error correctly. For example, if an error occurs in a backend application while processing a client request, often the only thing you can do is log the error and respond to the client with a 500 HTTP status code.

In such cases, wrapping errors in custom types can be a waste of effort or might even unnecessarily complicate the code. Instead of wrapping them, you can simply "bubble them up" as an anonymous trait object: `Box<dyn Error>`.

Let’s look at an example. We’ll write two functions that return two different error types, and a third function that calls them both and propagates any resulting errors as a `Box<dyn std::error::Error>>`.

```rust
#[derive(Debug, thiserror::Error)]
#[error("Error A")]
struct ErrA;

#[derive(Debug, thiserror::Error)]
#[error("Error B")]
struct ErrB;

fn fail_a() -> Result<(), ErrA> {
    Err(ErrA)
}
fn fail_b() -> Result<(), ErrB> {
    Err(ErrB)
}

fn fail_something(is_a: bool) -> Result<(), Box<dyn std::error::Error>> {
    if is_a {
        // The ? operator will repackage the error into the expected type - Box<dyn std::error::Error>
        // as if we had explicitly called:
        // fail_a().map_err(|e| Box::new(e) as Box<dyn std::error::Error>)
        let r = fail_a()?;
        Ok(r)
    } else {
        let r = fail_b()?;
        Ok(r)
    }
}

fn main() {
    if let Err(e) = fail_something(true) {
        println!("Underlying error: {}", e.to_string());
    }
    if let Err(e) = fail_something(false) {
        println!("Underlying error: {}", e.to_string());
    }
}
```

The program outputs:

```
Underlying error: Error A
Underlying error: Error B
```

Now we know how to return any error without worrying about its specific type. However, sometimes you still need to handle a specific type of error separately. This can be achieved using the [downcast_ref](https://doc.rust-lang.org/std/error/trait.Error.html#method.downcast_ref) method, which is defined for the `dyn Error` trait object.

```rust
# #[derive(Debug, thiserror::Error)]
# #[error("Error A")]
# struct ErrA;
# 
# #[derive(Debug, thiserror::Error)]
# #[error("Error B")]
# struct ErrB;
# 
# fn fail_a() -> Result<(), ErrA> {
#     Err(ErrA)
# }
# fn fail_b() -> Result<(), ErrB> {
#     Err(ErrB)
# }
# 
# fn fail_something(is_a: bool) -> Result<(), Box<dyn std::error::Error>> {
#     if is_a {
#         let r = fail_a()?;
#         Ok(r)
#     } else {
#         let r = fail_b()?;
#         Ok(r)
#     }
# }
# 
fn main() {
    if let Err(e) = fail_something(true) {
        if let Some(err_a) = e.downcast_ref::<ErrA>() {
            println!("Handle ErrA separately: {err_a}")
        } else {
            println!("Underlying error: {}", e.to_string());
        }
    }
}
```

## Anyhow

If you aren't thrilled about manually working with `Box<dyn std::error::Error>`, the Rust ecosystem offers the [anyhow](https://crates.io/crates/anyhow) crate, which simplifies working with "type-erased" (opaque) errors.

anyhow provides its own type-erased error type, [anyhow::Error](https://docs.rs/anyhow/latest/anyhow/struct.Error.html), and its own result type, [anyhow::Result\<T>](https://docs.rs/anyhow/latest/anyhow/type.Result.html), which is an alias for`std::result::Result<T, anyhow::Error>`.

To start, let's look at a simple program that demonstrates how anyhow integrates into the error-handling process.

```rust,noplayground
#[derive(Debug, thiserror::Error)]
#[error("My custom error")]
struct MyError;

fn fail_with_specific_error() -> Result<(), MyError> { // returns a specific error
    Err(MyError)
}

fn call_failable() -> anyhow::Result<()> { // returns a type-erased anyhow::Error
    let r = fail_with_specific_error()?;
    Ok(r)
}

fn main() {
    match call_failable() {
        Ok(_) => println!("It was fine"),
        Err(e) => {
            if let Some(_my_err) = e.downcast_ref::<MyError>() {
                eprintln!("It failed MyError")
            } else {
                eprintln!("It failed with: {}", e.root_cause())
            }
        }
    }
}
```

As you can see, in the body of the `call_failable` function, the specific error is automatically repacked into an `anyhow::Error`, much like how we repacked specific errors into `Box<dyn Error>` in the [previous section](error-handling.md#boxdyn-error).

So, what advantages does anyhow offer?

### backtrace

`anyhow::Error` doesn't just wrap an error; it can also capture a backtrace, allowing you to easily identify the exact location where the error occurred.

By default, backtraces are not captured (as it is a resource-intensive process). To enable them, you must set the `RUST_LIB_BACKTRACE=1` environment variable before running your program.

Let's rewrite the previous example to display a backtrace:

```rust,noplayground
#[derive(Debug, thiserror::Error)]
#[error("My custom error")]
struct MyError;

fn fail_with_specific_error() -> Result<(), MyError> {
    Err(MyError)
}

fn call_failable() -> anyhow::Result<()> {
    let r = fail_with_specific_error()?;
    Ok(r)
}

fn main() {
    match call_failable() {
        Ok(_) => println!("It was fine"),
        Err(e) => {
            eprintln!("It failed with: {}", e.root_cause());
            eprintln!("Backtrace:\n{}", e.backtrace());
        }
    }
}
```

Set the environment variable before running (in Windows CMD, this is done with the command `set RUST_LIB_BACKTRACE=1`):

```
$ export RUST_LIB_BACKTRACE=1
$ cargo run
It failed with: My custom error
Backtrace:
   0: anyhow::error::<impl core::convert::From<E> for anyhow::Error>::from
             at /home/stas/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/anyhow-1.0.100/src/backtrace.rs:27:14
   1: <core::result::Result<T,F> as core::ops::try_trait::FromResidual<core::result::Result<core::convert::Infallible,E>>>::from_residual
             at /home/stas/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/result.rs:2177:27
   2: test_rust::call_failable
             at ./src/main.rs:10:13
   3: test_rust::main
             at ./src/main.rs:15:11
   4: core::ops::function::FnOnce::call_once
             at /home/stas/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/core/src/ops/function.rs:250:5
   5: std::sys::backtrace::__rust_begin_short_backtrace
             at /home/stas/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/sys/backtrace.rs:158:18
   6: std::rt::lang_start::{{closure}}
             at /home/stas/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/rt.rs:206:18
   7: core::ops::function::impls::<impl core::ops::function::FnOnce<A> for &F>::call_once
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/core/src/ops/function.rs:287:21
   8: std::panicking::catch_unwind::do_call
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/panicking.rs:590:40
   9: std::panicking::catch_unwind
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/panicking.rs:553:19
  10: std::panic::catch_unwind
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/panic.rs:359:14
  11: std::rt::lang_start_internal::{{closure}}
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/rt.rs:175:24
  12: std::panicking::catch_unwind::do_call
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/panicking.rs:590:40
  13: std::panicking::catch_unwind
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/panicking.rs:553:19
  14: std::panic::catch_unwind
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/panic.rs:359:14
  15: std::rt::lang_start_internal
             at /rustc/ed61e7d7e242494fb7057f2657300d9e77bb4fcb/library/std/src/rt.rs:171:5
  16: std::rt::lang_start
             at /home/stas/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/rt.rs:205:5
  17: main
  18: __libc_start_call_main
             at ./csu/../sysdeps/nptl/libc_start_call_main.h:58:16
  19: __libc_start_main_impl
             at ./csu/../csu/libc-start.c:360:3
  20: _start
```

In the second element of the backtrace chain, you can see that the specific error was repacked into `anyhow::Error` at line `./src/main.rs:10:13`.

### context

Sometimes the original error message is not very informative. The `anyhow` crate allows you to attach additional textual information to an error — a context. This context can later be retrieved simply by calling `to_string()` on the `anyhow::Error` object.

To add context to an error, use the `.context()` method on the result (a `Result` value).

```rust,noplayground
fn my_func() -> anyhow::Result<Тип> {
    let result = func_that_can_fail()
            .context("informative description")?;
    Ok(result)
}
```

Let’s look at an example. We will write a program that attempts to read a non-existent file and use context to specify which file could not be read.

```rust,noplayground
use anyhow::Context;

fn read_non_existing_file() -> anyhow::Result<String> {
    let text = std::fs::read_to_string("non_existing_file.txt")
        .context("Cannot read non_existing_file.txt")?;
    Ok(text)
}

fn main() {
    match read_non_existing_file() {
        Ok(text) => println!("File content: {text}"),
        Err(e) => {
            eprintln!("Failed with error: {}", e.root_cause());
            eprintln!("Error context: {}", e.to_string());
        }
    }
}
```

Run:

```
$ cargo run
Failed with error: No such file or directory (os error 2)
Error context: Cannot read non_existing_file.txt
```

As you can see, without using context we would only get a rather uninformative error message: “No such file or directory (os error 2)”. By adding context, however, we can specify exactly which file is missing.

## General Recommendations for Error Handling

* When writing a library, it is recommended to use specific and detailed error types. To simplify the creation of error types, it is recommended to use thiserror.
* When writing an end-user application, in areas where it is not possible to properly handle each error type separately, use anyhow, as it greatly simplifies error handling.


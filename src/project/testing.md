# Testing

## Unit Testing

In Rust, it is idiomatic to write unit tests in the same file as the functions they are testing.

A unit test is a function marked with the `#[test]` attribute.

For example, let's say we have a program that increments a number using an `inc` function:

```rust
fn inc(a: i32) -> i32 {
    a + 1
}

fn main() {
    let a = 5;
    let b = inc(a);
    println!("{b}");
}
```

Now, let's write a couple of unit tests for the `inc` function: `test_inc_1` and `test_inc_2`.

```rust
fn inc(a: i32) -> i32 {
    a + 1
}

#[test]
fn test_inc_1() {
    assert_eq!(inc(1), 2);
}

#[test]
fn test_inc_2() {
    assert_eq!(inc(7), 8);
}

fn main() {
    let a = 5;
    a = inc(a);
    println!("{a}");
}
```

The `assert_eq!` macro compares its arguments. If they are not equal, the macro triggers a panic, causing the test to fail.


***

To run all tests in a package, use the `cargo test` command:

```
$ cargo test
running 2 tests
test my_module::test_inc_1 ... ok
test my_module::test_inc_2 ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

You can run a specific test by name like this:

```
$ cargo test -- --exact my_module::test_inc_1
```

> [!NOTE]
> From an organizational perspective, unit tests are compiled into a separate executable binary found in the `target/debug/deps/` directory. It follows the naming convention `crate_name-hash`. This binary contains the entire contents of the module being tested, the test functions themselves, and the test runner code used to generate the report.

## Integration Tests

While unit tests are typically located within the same modules as the functions they test, integration tests reside in a separate directory at the root of your package: tests. Consequently, integration tests can only access the public API of the crate being tested.

```
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── module1.rs
│   └── module2.rs
└── tests/
    ├── integration-tests_1.rs
    └── integration-tests_2.rs
```

Unlike unit tests, every file in `tests/*.rs` is compiled into a <ins>separate executable</ins>. Essentially, each integration test file acts as its own individual test crate.

Because of this, it makes sense to group integration tests based on the resources they require. For example, you might place all tests requiring a relational database in one file and all tests requiring a distributed cache in another. This helps manage setup overhead and compilation times.

***

Let’s look at an example to better understand the testing structure. Our target for integration testing will be a library crate with two modules:

* `data` — functionality for interacting with storage.
* `user_service` — functionality for handling user records.

First, create a new project:

```
cargo new test_project --lib
```

File `src/data.rs`:

```rust,noplayground
// Represents a user stored in some data store.
pub struct User {
    pub first_name: String,
    pub last_name: String,
}

// Declares an interface for retrieving a user object from storage.
// The specific implementation for a specific storage engine is 
// left to the library user, who must implement this trait.
pub trait DataSource {
    // Retrieves a user with the specified ID from storage
    fn find_user_by_id(&self, id: u64) -> Option<User>;
}
```

File `src/user_service.rs`:

```rust,noplayground
use crate::data::DataSource;

// A function that takes a storage implementation and a user ID,
// then returns the user's full name.
pub fn get_user_full_name(ds: &dyn DataSource, user_id: u64) -> Option<String> {
    ds.find_user_by_id(user_id)
      .map(|user| format!("{} {}", user.first_name, user.last_name))
}
```

File `src/lib.rs`:

```rust,noplayground
pub mod data;
pub mod user_service;
```

---

Now, let’s write an integration test for the `get_user_full_name` function.

This function requires an implementation of the `DataSource` trait. In a production environment, this would likely be an integration with a real database or an identity service. However, for testing purposes, we will create a Mock (a stub) that returns predefined test data.

File `tests/user_service_test.rs`:

```rust,noplayground
use test_project::{data::{DataSource, User}, user_service::get_user_full_name};

// Mock implementation for testing
struct DataSourceMock;

impl DataSource for DataSourceMock {
    fn find_user_by_id(&self, id: u64) -> Option<User> {
        Some(User { first_name: "John".to_string(), last_name: "Doe".to_string() })
    }
}

#[test]
fn test_get_user_full_name() {
    let result = get_user_full_name(&DataSourceMock, 1);
    assert_eq!(Some("John Doe".to_string()), result);
}
```

Now, run the test:

```
$ cargo test
     Running tests/user_service_test.rs (target/debug/deps/user_service_test-9f98ad2948b56b39)

running 1 test
test test_get_user_full_name ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

The test passes!

***

> [!TIP]
> If certain dependencies (libraries) are only needed for integration tests (e.g., Testcontainers), you can add them to the `[dev-dependencies]` section in your `Cargo.toml`. These dependencies will not be included in the final production build.

At this stage, there's no need to dive deeper into integration tests. We will explore them in more detail in the [Testcontainers](../web/testcontainers.md) chapter when we discuss backend development.

## cargo-nextest

If you find yourself needing more flexibility than the standard cargo test runner provides, check out [nextest](https://nexte.st/). This is an alternative test runner that offers:

* Cleaner, more informative output during test execution.
* The ability to run lightweight tests in parallel while keeping heavy tests serial.
* Built-in support for test retries (great for flaky integration tests).
* Report generation in JUnit XML format.
* And much more.

To install nextest, run:

```
cargo install cargo-nextest --locked
```

Once installed, you can run your tests using:

```
cargo nextest run
```

Try it out and compare the output of `cargo test` and `cargo nextest`.

test:

```
$ cargo test
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.21s
     Running unittests src/main.rs (target/debug/deps/test_rust-ba9e2d97573eceb4)

running 3 tests
test test_1 ... ok
test test_2 ... ok
test test_3 ... ok

test result: ok.
3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

nextest:

```
$ cargo nextest run
    Finished `test` profile [unoptimized + debuginfo] target(s) in 0.05s
────────────
 Nextest run ID 06dccce6-29f5-45a0-9ec4-1d9328eaae82 with nextest profile: default
    Starting 3 tests across 1 binary
        PASS [   0.006s] test_rust::bin/test_rust test_3
        PASS [   0.006s] test_rust::bin/test_rust test_2
        PASS [   0.006s] test_rust::bin/test_rust test_1
────────────
     Summary [   0.007s] 3 tests run: 3 passed, 0 skipped
```


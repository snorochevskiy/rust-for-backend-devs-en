# Workspace Project

Typically, large projects are broken down into subprojects.

For example, when writing back-end applications, functionality is often divided into layers:

* Data Storage Layer: Database connections, functions with SQL queries, and data structures.
* Business Logic Layer: Functions and structures that implement the program's core logic.
* Presentation Layer: REST/gRPC endpoints, WebSockets, etc.

Layers solve more than just the problem of splitting functionality; they also address the logical isolation of these subsets from one another. For instance:

* The data storage layer should not "see" functionality from other modules.
* The business logic layer should "see" the entities stored in the database and the data access functions, but it should not see the specific implementation of the storage. It also shouldn't see the presentation layer.
* The presentation layer should know about both the data and business functionality, but without their implementation details.

![](img/workspace_layered_project.svg)

In many languages, such as Java, this type of project consisting of separate subproject layers is called a "multi-module project". However, since the word "module" refers to a different entity in Rust, these are called **workspace projects**.

## Workspace

A **.workspace**. is a type of package that contains other packages instead of code.

```
workspace-package/
  ├── Cargo.toml
  ├── пакет-1/
  │   ├── Cargo.toml
  │   └── src/
  │       └── lib.rs
  └── пакет-2/
      ├── Cargo.toml
      └── src/
          └── main.rs
```

The `Cargo.toml` file in the root directory looks like this:

```toml
[workspace]
members = ["пакет-1", "пакет-2"]
```

When you run Cargo (e.g., `cargo build`) in the root workspace directory, Cargo reads the `Cargo.toml`, recognizes it is a workspace project, and performs the build for all child packages while correctly resolving dependencies between them.

## Workspace Project Example

As an example, let’s create a simple program that prints specific text and the word count of that text.

We will organize the program as a workspace project consisting of three packages:

* package 1: data (library) – functionality for retrieving text.
* package 2: processor (library) – functionality for counting the number of words.
* package 3: cli (executable) – prints the text to the console.

Create a new directory named `workspace_test` for our project.

Inside this directory, create a `Cargo.toml` file with the following content:

```toml
[workspace]
members = [ ]
```

Now, create the child packages by running the following commands inside `workspace_test`:

```sh
cargo new data --lib
cargo new processor --lib
cargo new cli --bin
```

Cargo will automatically update the root `Cargo.toml`, which should now look like this:

```toml
[workspace]
members = ["cli", "processor","data"]
```

The project's file tree should look like this:

```
workspace_test/
  ├── Cargo.toml
  ├── data/
  │    ├── Cargo.toml
  │    └── src/
  │         └── lib.rs
  ├── processor/
  │    ├── Cargo.toml
  │    └── src/
  │         └── lib.rs
  └── cli/
       ├── Cargo.toml
       └── src/
            └── main.rs
```

---

First, write the code for the library that provides the text. In `data/src/lib.rs`:

```rust,noplayground
const TEXT: &str = "One two three four five six seven eight nine ten.";

pub fn get_text() -> String {
    TEXT.to_string()
}
```

***

Next, add a dependency on the `data` package in `processor/Cargo.toml`:

```toml
[package]
name = "processor"
version = "0.1.0"
edition = "2024"

[dependencies]
data = { path = "../data" }
```

Now you can use functions imported from `data` within the `processor` package. Let's write a function that returns the text and its word count. In `processor/src/lib.rs`:

```rust,noplayground
use data;

pub fn get_text_with_info() -> (String, usize) {
    let text = data::get_text();
    let words_count = text.split(" ").count();
    (text, words_count)
}
```

***

Next add a dependency on `processor` in the `cli` package.

```toml
[package]
name = "cli"
version = "0.1.0"
edition = "2024"

[dependencies]
processor = { path = "../processor" }
```

Finally, created an executable that prints to the console a text along with the number of words in the text. File `cli/src/main.rs`:

```rust,noplayground
use processor;

fn main() {
    let (text, words_count) = processor::get_text_with_info();
    println!("Text: {text}");
    println!("Words count: {words_count}");
}
```

***

Everything is ready. We can build and run our program:

```
$ cargo build
   Compiling data v0.1.0 (/home/user/project/rust/workspace_test/data)
   Compiling processor v0.1.0 (/home/user/project/rust/workspace_test/processor)
   Compiling cli v0.1.0 (/home/user/project/rust/workspace_test/cli)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.28s

$ ./target/debug/cli 
Text: One two three four five six seven eight nine ten.
Words count: 10

```

Note that in a workspace project, we run the program from the `target` directory located at the project root, not from `cli/target`.

> [!NOTE]
> In a workspace project, you can also execute Cargo commands for a specific package using the `-p` option. For example, to run `cargo run` for the `cli` package:
> 
> ```
> $ cargo run -p cli
>     Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.02s
>      Running `target/debug/cli`
> Text: One two three four five six seven eight nine ten.
> Words count: 10
> ```

## Workspace Dependencies

Since a workspace project has multiple packages, it is common for several packages to require the same library as a dependency (e.g., `rand`).

This requires careful management to ensure that versions are updated consistently across all packages.

To solve this, you can define dependency versions in the root workspace `Cargo.toml` and reference them in the child packages.

First, declare the library version in the root `Cargo.toml`:

```toml
[workspace]
members = ["child_package"]

[workspace.dependencies]
rand = "0.8"
```

Now, in the child package, you can include the `rand` dependency like this:

```toml
[package]
name = "child_package"
version = "0.1.0"
edition = "2024"

[dependencies]
rand = { workspace = true }
```

Updating the dependency version in the root `Cargo.toml` will now be automatically reflected in all child packages.

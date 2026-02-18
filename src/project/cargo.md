# Cargo

Using the `rustc` compiler directly is fine for building small test programs; however, when writing a real application that uses third-party libraries, a build system is indispensable.

In most programming languages, the build system is shipped separately from the language toolchain itself. For example, in Java, Maven and Gradle are popular tools distributed separately from the JDK. But in the case of Rust, the `rustup` utility installs the build system along with the compiler. This build system is called Cargo.

Cargo is capable of:

* Creating a new Rust project
* Building an executable file or library
* Automatically downloading and compiling dependencies (libraries)
* Running unit and integration tests
* Running benchmarks and aggregating the results
* Downloading and building various utilities from the Rust ecosystem

## Creating a Project

To create a new Rust project managed by the Cargo build system, you need to execute the command `cargo new project_name`.

For example, let's create a "Hello World" project:

```
cargo new hello_world --bin
```

The `--bin` option indicates that we want to create an executable program (this is the default project type). If we wanted to create a library, we would specify the `--lib` option instead.

After executing the command, Cargo will create the following directory tree:

```
hello_world/
├── Cargo.toml
└── src/
    └── main.rs
```

This is the standard Cargo project structure:

* Source code is located inside the `src` directory.
  * If we are creating an executable program, the main file is `main.rs`.
  * If we are creating a library, the main file is lib.rs.
* `Cargo.toml` is the project configuration file where Cargo retrieves the main settings.

Cargo will populate `src/main.rs` with a "boilerplate" Hello World program:

```rust
fn main() {
    println!("Hello, world!");
}
```

## Cargo.toml

The `Cargo.toml` file contains the project configuration in TOML (Tom's Obvious, Minimal Language).

Immediately after creation, our `Cargo.toml` should look something like this:

```toml
[package]
name = "hello_world"
version = "0.1.0"
edition = "2024"

[dependencies]
```

The `package` section contains basic information about the project:

* `name` — the name of the crate (we will talk about these later). By default, the name of the executable file will be the same as this name.
* `version` — the version of the program or library.
* `edition` — the version of the Rust language that the program code corresponds to. [Learn more](https://doc.rust-lang.org/edition-guide/editions/).

The `dependencies` section is used to specify external dependencies (libraries) for the program. We will discuss these later.

## Building and Running

To build the executable file, use the following command:

```
cargo build
```

After this, the executable file should appear in the target/debug subdirectory. Let's run it to make sure it works:

On Linux/Mac:
```
$ ./target/debug/hello_world 
Hello, world!
```

On Windows:
```
target\debug\hello_world 
Hello, world!
```

As you might have guessed from the directory name `target/debug`, Cargo builds the executable with debug settings by default. If we need an optimized release build without debug information, we must pass the `--release` flag to the build command:

```
cargo build --release
```

The resulting executable will appear in the `target/release` directory.

---

Additionally, Cargo has a command that combines building and running into one step — `run`:

```
$ cargo run
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.01s
     Running `target/debug/hello_world`
Hello, world!
```

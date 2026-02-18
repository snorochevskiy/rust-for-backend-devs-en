# Creating a Library

We already know that to create a Rust project that compiles into an executable binary, we use the `cargo new` command. To create a library, the same command is used, but with the `--lib` flag:

```
cargo new my_lib --lib
```

The `--lib` flag indicates that `src/lib.rs` should be created instead of `src/main.rs`.

```
├── Cargo.toml
└── src/
    └── lib.rs
```

(In fact, there is nothing stopping you from having both `src/main.rs` and `src/lib.rs` simultaneously, and this is often done.)

Now, in the `[package]` section of `Cargo.toml`, we need to specify what type of library we are making. For this, the `crate-type` field is used:

```toml
[package]
name = "my_lib"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["lib"]

[dependencies]
```

The following options are available for `crate-type`:

* **`lib`** — A library for a Rust program. This means the project can be compiled into various types of libraries depending on how the program using the library is built. The only guarantee is that the library will be functional for any other Rust project.
* **`rlib`** — A static library specific to Rust programs, represented by a file with the `*.rlib` extension.
* **`dylib`** — A dynamic library (`*.dll` on Windows, `*.so` on Linux, `*.dylib` on MacOS) that is suitable only for Rust programs.
* **`cdylib`** — A system dynamic library that can be used by programs written in other languages.
* **`staticlib`** — A static library (`*.a` or `*.lib`). It is suitable for static linking into a program written in a language other than Rust.
* **`proc-macro`** — A library containing a procedural macro.

As you can see, in the general case of developing a library for other Rust applications, the optimal choice is `crate-type=["lib"]`.

All that's left is to write the code for our library in `src/lib.rs`. For our purposes, anything will do — for example, a function that adds two numbers:

```rust
pub fn sum2(a: i32, b: i32) -> i32 {
    a + b
}
```

Note that functions exported from a library must be marked with the `pub` keyword.

## Using a Library

As we know, when we specify a library in the `[dependencies]` section of `Cargo.toml`, Cargo attempts to find the corresponding crate on crates.io.

However, Cargo can download dependencies from more than just crates.io:

*   git репозиториев
    ```toml
    [dependencies]
    crate_name = { git = "https://github.com/account/repository.git", branch = "main" }
    ```
*   локальной файловой системы
    ```toml
    [dependencies]
    crate_name = { path = "/path/to/code/" }
    ```
*   альтернативных репозиториев
    ```toml
    [dependencies]
    crate_name = { version = "1.0", registry = "repository" }
    ```

In the same directory where we created the `my_lib` library project, let's create a new project that will use our library as a dependency.

```
cargo new lib_usage --bin
```

Open `Cargo.toml` for the newly created project and add the dependency on my_lib by specifying the path to its directory.

```toml
[package]
name = "lib_usage"
version = "0.1.0"
edition = "2024"

[dependencies]
my_lib = { path = "../my_lib" }
```

Now, in `src/main.rs`, let's create a simple example of using our library.

```rust
use my_lib::sum2;

fn main() {
    println!("sum2(1,2)={}", sum2(1, 1));
}
```

And run it:

```
lib_usage$ cargo run
     Locking 1 package to latest Rust 1.89.0 compatible version
   Compiling my_lib v0.1.0 (/home/user/projects/rust/my_lib)
   Compiling lib_usage v0.1.0 (/home/user/projects/rust/lib_usage)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.23s
     Running `target/debug/lib_usage`
sum2(1,2)=2
```

In the build logs, you can see how the dependency `my_lib` is compiled first, followed by our application itself.

# First Look

To kick things off and lay the foundation for our understanding of the language, let’s take a look at what a "Hello World" program looks like in Rust.

```rust
// This is a comment
fn main() {
    println!("Hello world!");
}
```

As you can see, Rust is a language with what is known as C-style syntax: the function body is wrapped in curly braces, and each expression ends with a semicolon.

Just like in many C-style languages, the `main` function is the entry point—the execution of the program begins here.

---

Let’s get this code running.

1\) Save the program text to a file named `main.rs` in a directory of your choice.

2\) Open your terminal and navigate (using the `cd` command) to the directory containing `main.rs`.

3\) Compile `main.rs` using the Rust compiler, `rustc`:

```
rustc main.rs
```

The compiler should generate an executable file: `main.exe` on Windows or `main` on Linux/macOS.

4\) Run the program: 

On Linux or macOS:

```
./main
Hello world!
```

On Windows:

```
main
Hello world!
```


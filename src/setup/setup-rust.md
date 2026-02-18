# Installing Rust

To work with Rust, you will need at minimum:

* The Rust toolchain (compiler, build system, etc.) — to compile Rust source code files into object files (files containing machine code)
* A linker — to combine multiple object files into a final executable program
* The C standard library

The need for the C standard library arises from the fact that Rust uses C functions for memory allocation as well as for certain memory copy operations.

Building a Rust application roughly looks like this:

![](img/rust_compilation.svg)

In other words, when producing the final executable from compiled modules, the linker must have access to the C standard library.

***

The Rust toolchain is installed using the official utility [rustup](https://www.rust-lang.org/tools/install), while installing the linker and the C standard library depends on the operating system and the C++ toolchain being used.

In the following chapters, we will cover installing the Rust and C++ toolchains on Windows and Linux.

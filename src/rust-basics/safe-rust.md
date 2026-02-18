# Safe Rust

Before we dive into learning specific language constructs, it is important to say that Rust consists of two subsets: Safe Rust and Unsafe Rust.

By default, we write code in Safe Rust, where the compiler provides guaranteed protection against:

* Memory leaks
* Data races in multi-threaded environments
* Segmentation faults / null pointer access errors
* Undefined behavior

For these safety guarantees, we pay with a certain degree of freedom. Specifically, in Safe Rust we cannot:

* Manipulate memory using raw pointers
* Call code from libraries written not in Rust (FFI)
* Work with potentially unsynchronized data

These operations belong to the Unsafe subset of Rust and can only be performed within a special `unsafe` block.

But please don't be confused by this "Unsafe". When writing back-end applications (the primary focus of this book), you will rarely need to use Unsafe Rust. In most applications, it can be avoided entirely.

Throughout our journey, we will be highlighting actions that are available only in Unsafe Rust, so you could see how rare they are.

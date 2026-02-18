# Safe Rust

Before we dive into learning specific language constructs, it is important to note that Rust consists of two subsets: Safe Rust and Unsafe Rust.

By default, we write code in Safe Rust, where the compiler provides guaranteed protection against the following:

* Memory leaks
* Data corruption due to multiple access (concurrent modification errors)
* Data races in multi-threaded environments
* Segmentation faults / null pointer access errors
* Undefined behavior

For these safety guarantees, we pay with a certain degree of freedom. Specifically, a range of operations is prohibited in Safe Rust, such as:

* Manipulating memory using raw pointers
* Calling code from libraries not written in Rust (FFI)
* Working with potentially unsynchronized data

These operations belong to the Unsafe subset of Rust and can only be performed within a special `unsafe` block.

Don't let the term "Unsafe" alarm you. When writing back-end applications (the primary focus of this book), you will rarely need to use Unsafe Rust. In most applications, it can be avoided entirely.

Throughout our journey, we will occasionally point out actions available only in Unsafe Rust. This will allow you to see for yourself just how rare those cases actually are.

# Pointers

## unsafe

Rust features pointers that function almost identically to those in C. However, as mentioned at the beginning of this book, working with raw pointers is forbidden in "Safe Rust." This is why we will be actively using the `unsafe` keyword throughout this chapter.

We previously encountered the unsafe block in the section on [Static Variables](./variables.md#static):

```rust
unsafe {
    // code
}
```

Inside such a block, you can perform unsafe operations, including working with raw pointers.

You can also mark an entire function with the `unsafe` keyword:

```rust
unsafe fn my_function() {
    ...
}
```

In this case, the entire body of the function becomes an unsafe block. Furthermore, an unsafe function can only be called from within another unsafe block or another unsafe function.

## Working with Pointers

Just like in C, a pointer in Rust is a variable that stores a memory address. At the source code level, a pointer has a type representing the data it points to.

A pointer type is constructed by adding the prefix `*const` (for immutable pointers) or `*mut` (for mutable pointers) before the type of the value being referenced.

For example, if a variable is of type `i32`, an immutable pointer to it will be `*const i32`.

```rust
let p1: *const i32;  // Immutable pointer to an i32
let p2: *mut i32;    // Mutable pointer to an i32
let p3: *mut String; // Mutable pointer to a String
```

There are several ways to obtain a pointer to an object:

1\) Casting from a reference

```rust
let mut v: i32 = 5;
let const_ptr: *const i32 = &v as *const i32;
let mut_ptr:   *mut i32   = &mut v as *mut i32;
```

2\) Using `&raw` (this succeeded the `addr_of` and `addr_of_mut` macros)

```rust
let mut v: i32 = 5;
let const_ptr: *const i32 = &raw const v;
let mut_ptr:   *mut i32   = &raw mut v;
```

3\) The [addr_of](https://doc.rust-lang.org/stable/std/ptr/macro.addr_of.html) and [addr_of_mut](https://doc.rust-lang.org/stable/std/ptr/macro.addr_of_mut.html) macros (the older standard library method).

```rust
let mut v: i32 = 5;
let const_ptr: *const i32 = std::ptr::addr_of!(v);
let mut_ptr:   *mut i32   = std::ptr::addr_of_mut!(v);
```

To dereference a pointer (access the value at the address), use the `*` operator, just as in C.

Consider this simple example:

```rust
fn main() {
    let a = 5;
    let ptr = (&a) as *const i32; // Take a reference and cast it to a pointer
    unsafe {
        // Dereference the pointer to get the variable's value
        println!("{}", *ptr); // 5
    }
}
```

Note: Casting a reference to a pointer is a safe operation and can be done outside an `unsafe` block. However, <ins>dereferencing</ins> a pointer or converting a pointer back into a reference requires `unsafe`.

## Bypassing Reference Restrictions

Recall that in Rust, at any given point in a program, you can have either one mutable reference or any number of immutable references to an object. In the vast majority of cases—especially when writing backend logic — this restriction is not a hindrance. However, when implementing data structures or specific algorithms, you often need more than one mutable reference to the same object.

For example, in a doubly linked list, you simultaneously need two mutable references to the same elements: one from the "head" direction and one from the "tail" direction.

![](img/pointers_list.svg)

Another example is merge sort, which involves splitting the original sequence into segments. Each segment is sorted independently and must therefore have its own mutable reference.

An `unsafe` block does not allow you to directly violate reference safety rules. However, `unsafe` allows you to create an additional reference through an intermediate pointer:\
Mutable Reference -> Pointer -> Another Mutable Reference.

```rust
fn main() {
    let mut a = 5;
    unsafe {
        let r1: &mut i32 = &mut a; // First mutable reference
        let ptr: *mut i32 = r1 as *mut i32; // Mutable pointer
        let r2: &mut i32 = ptr.as_mut().unwrap(); // Pointer into a second reference
        inc(r1);
        inc(r2);
    }
    println!("{a}"); // 7
}

fn inc(a: &mut i32) {
    *a = *a + 1;
}
```

Naturally, this technique should only be used as a last resort. It is also highly recommended to:

* Thoroughly cover code using `unsafe` with tests.
* Diagnose your code using[Miri](https://github.com/rust-lang/miri) — an interpreter for Rust that detects undefined behavior in unsafe code.


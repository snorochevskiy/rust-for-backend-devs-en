# Smart Pointers

Until now, we have mostly worked explicitly with data stored on the stack. We have, of course, seen many examples using vectors and strings that store their elements on the heap; however, these types encapsulate heap management internally, completely hiding the details from us. In this chapter, we will explicitly explore working with the heap using smart pointers.

> [!NOTE]
> "Smart Pointer" is a term originating from C++. Unlike C, where a pointer is simply a memory cell storing an address, a smart pointer is a class that doesn't just provide access to data via an address but also knows how to automatically clean up the memory it references.

## Box

The first pointer we will examine is [Box](https://doc.rust-lang.org/std/boxed/struct.Box.html). We previously mentioned it briefly in the [Returning a Trait from a Function](traits.md#return-impl-trait) section.

`Box<T>` is a generic type that stores the address of a value of type `T` allocated on the heap. `Box` owns the data on the heap; therefore, when the `Box` variable goes out of scope, the corresponding heap memory is automatically deallocated.

> [!TIP]
> Drawing an analogy with C++, `Box` is the direct equivalent of the `unique_ptr` smart pointer.

In terms of memory layout, `Box` is a so-called "zero-cost abstraction." This means it is simply a cell containing an address located on the stack, and nothing more.

<pre class="ascii-diagram">
┌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┐    ┌╌╌╌╌╌╌╌╌┐
┆Stack            ┆    ┆Heap    ┆
┆ ┌────────────┐  ┆    ┆ ┌────┐ ┆
┆ │Box: pointer├────────>│Data│ ┆
┆ └────────────┘  ┆    ┆ └────┘ ┆
└╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┘    └╌╌╌╌╌╌╌╌┘
</pre>

The simplest way to create a`Box` is to use the `Box::new(T)` constructor, which takes a value as an argument and moves that value onto the heap.

Let's look at an example:

```rust
// Structure for storing 2D point coordinates
struct Point2D { x: i32, y: i32 }

fn main() {
    let p: Point2D = Point2D {x: 5, y: 2}; // Create value on the stack
    let b: Box<Point2D> = Box::new(p);     // Move value to the heap
}
```

Why is storing values on the heap useful? The reason is that for a value to be stored on the stack, its size must be known at compile time. For example, with the `Point2D` struct, the size of the value will always be the same (two `i32` fields). However, the total size of structures like a vector is not known at compile time because the number of elements in the vector is unknown.

For clarity, let's write a classic data structure: a singly linked list.

<pre class="ascii-diagram">
┌──────────┐    ┌──────────┐    ┌──────────┐
│ value 1  │ ╭─>│ value 2  │ ╭─>│ value   3│
├──────────┤ │  ├──────────┤ │  ├──────────┤
│next: ptr ├─╯  │next: ptr ├─╯  │next: nil │
└──────────┘    └──────────┘    └──────────┘
</pre>

At first glance, this construction could be described like this:

```rust,noplayground
// A list is:
enum List<T> {
    Nil,              // either an empty list
    Elem(T, List<T>), // or a pair: value + list
}
```

However, the compiler will issue the following error:

```rust,noplayground
enum List<T> {
    Nil,
    Elem(T, List<T>),
}//         ------- recursive without indirection
// error[E0072]: recursive type `List` has infinite size
// insert some indirection (e.g., a `Box`, `Rc`, or `&`) to break the cycle
//   Elem(T, Box<List<T>>),
//           ++++       +
```

This is exactly what we discussed earlier: a recursive structure has an indeterminate size, making it impossible to place on the stack. The compiler kindly suggests using `Box`, which solves our problem perfectly:

```rust
#[derive(Debug)]
enum List<T> {
    Nil,
    Elem(T, Box<List<T>>),
}

use List::*;

fn main() {
    let list: List<i32> =
        Elem(1, Box::new(
            Elem(2, Box::new(
                Elem(3, Box::new(Nil))
        ))));
    println!("{:?}", list); // Elem(1, Elem(2, Elem(3, Nil)))
}
```

## The Deref and DerefMut Traits

For an object of type `Box`, you can use the dereference operator `*` as if it were a regular reference. Additionally, using the `&` operator on a `Box` object allows you to obtain a <ins>direct</ins> reference to the data on the heap.

```rust
fn main() {
    let mut b = Box::new(1);
    *b = 2;
    println!("{b}"); // 2

    increment(&mut b);
    println!("{b}"); // 3
}

fn increment(i: &mut i32) {
    *i += 1;
}
```

This behavior of the `Box` object (acting like a reference) is possible because the `Box` type implements the [Deref](https://doc.rust-lang.org/std/ops/trait.Deref.html) trait.

```rust,noplayground
pub trait Deref {
    type Target: ?Sized;
    fn deref(&self) -> &Self::Target;
}
```

This trait allows a type to provide a reference. Naturally, in the case of `Box`, it returns a reference to its data on the heap.

If a type implements the `Deref` trait, the compiler replaces `&object` with `object.deref()`.

As we might have noticed, the `deref` method returns an immutable reference. For cases where we want to be able to modify the value via a reference, there is the [DerefMut](https://doc.rust-lang.org/std/ops/trait.DerefMut.html) trait.

```rust,noplayground
pub trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```

The call `*object = value` is replaced by `*(object.deref_mut()) = value`.

## Rc — Shared Ownership { #rc }

Rust's ownership concept does not allow shared ownership of the same object; however, there are many data structures where this is necessary (e.g., a doubly linked list). For such situations, the Rust standard library provides the [Rc](https://doc.rust-lang.org/std/rc/struct.Rc.html) (Reference Counted) (Reference Counted) smart pointer.

Unlike `Box<T>`, which is essentially just a pointer to data on the heap, Rc<T> is a structure consisting of two fields:

* A pointer to the data on the heap.
* A pointer to the reference counter of the `Rc` object.

Consider a simple example of using `Rc`:

```rust
use std::rc::Rc;

fn main() {
    let rc1 = Rc::new("Hello".to_string());
    let rc2 = rc1.clone();
}
```

When we create a new object using `Rc::new(value)`:

1. An `Rc` structure object is created on the stack.
2. Space is allocated on the heap for the data, and the value passed to `Rc::new()` is moved into it. Then, the address of this value on the heap is assigned to the `Rc` object's "data pointer" field.
3. Space is allocated on the heap for the reference counter and initialized to one. The address of this counter is assigned to the `Rc` object's "counter pointer" field.

When we clone an `Rc` object:

1. A new `Rc` object is created on the stack.
2. The data pointer value is copied from the `Rc` object being cloned.
3. The reference counter pointer value is copied from the `Rc` object being cloned, and the counter itself is incremented.

When a variable storing an Rc object goes out of scope:

1. The `Rc` reference counter is decremented by `1`.
2. If the counter value becomes `0`, the memory where the data is stored and the memory where the counter itself is stored are both deallocated.

The layout of `Rc` in memory looks something like this:

![](img/smart_ponter_rc.svg)

> [!TIP]
> Drawing an analogy with C++, `Rc` is the direct equivalent of the `shared_ptr` smart pointer.

## Cell

The primary drawback of `Rc` is that, unlike `Box`, it does not implement the `DerefMut` trait, which means it doesn't allow you to modify its contents.

```rust,noplayground
use std::rc::Rc;

fn main() {
    let mut rc1 = Rc::new(1);
    *rc1 += 1;
 // ^^^^^^^^^ cannot assign
 // trait `DerefMut` is required to modify through a dereference,
 // but it is not implemented for `Rc<i32>`
}
```

Why is it designed this way? Let's recall the [Rust reference safety rule](ownership.md#referential-safety): you can have either one mutable reference or any number of immutable references to an object at the same time. This rule exists because, in the general case—without additional synchronization mechanisms—it is impossible to guarantee the validity of an immutable reference after the data has been modified via a mutable reference.

This is precisely why `Rc` allows multiple owners of the same object (analogous to multiple immutable references) but provides no way to change the value.

As noted above, having simultaneous immutable and mutable references is unsafe WITHOUT additional synchronization. The Rust standard library provides a synchronization mechanism specifically for these situations: the [Cell](https://doc.rust-lang.org/std/cell/struct.Cell.html) wrapper structure.

`Cell<T>` is a wrapper that allows you to replace its entire content safely and atomically, while specifically NOT allowing:

* Obtaining a <ins>mutable reference</ins> to its content, which prevents data corruption.
* Obtaining an <ins>immutable reference</ins>, which could become invalid if the value in the Cell were replaced.

To work with `Cell`, three main methods are typically used:

* [new(value)](https://doc.rust-lang.org/std/cell/struct.Cell.html#method.new) — creates a new Cell and initializes it with the given value.
* [replace(value)](https://doc.rust-lang.org/std/cell/struct.Cell.html#method.replace) — atomically places a new value into the Cell and returns the old value.
* [set(value)](https://doc.rust-lang.org/std/cell/struct.Cell.html#method.set) — atomically places a new value into the Cell and drops the old value.

Consider this example of using `Cell`:

```rust
use std::cell::Cell;

fn main() {
    let cell = Cell::new("aaa".to_string());

    // When replacing with a new value, the previous one is returned
    let old_string = cell.replace("bbb".to_string());
    println!("{old_string}");

    // If we don't need the previous value, we can simply overwrite it
    cell.set("ccc".to_string());
}
```

> [!NOTE]
> Notice that the `cell` variable is declared without the `mut` modifier, yet we were able to replace the stored string with another. This is possible because the replacement happens safely and atomically; therefore, the variable doesn't need to have "mutable" semantics with all its associated restrictions.

If a type implements the `Copy` trait, you can extract a copy of the value from the `Cell` using the [get](https://doc.rust-lang.org/std/cell/struct.Cell.html#method.get) method.

```rust
use std::cell::Cell;

fn main() {
    let cell = Cell::new(1);
    println!("{}", cell.get()); // c1 = 1
}
```

---

Now, by using the combination `Rc<Cell<T>>`, we can create data structures that require both shared ownership and the ability to replace the stored value.

```rust
use std::{cell::Cell, rc::Rc};

fn main() {
    let rc1 = Rc::new(Cell::new(1));
    let rc2 = rc1.clone();

    // Get a reference to the shared Cell and set a new value
    rc2.as_ref().set(5);

    println!("{:?}", rc1); // Cell { value: 5 }
}
```

> [!IMPORTANT]
> It is important to note that `Cell` only protects against scenarios where an immutable reference is corrupted by data modification via a mutable reference within a single thread. `Cell` does <ins>not</ins> provide any protection against data races.

## RefCell

An obvious limitation of `Cell` is that it only allows you to replace the stored value, not modify it in place. The [RefCell\<T\>](https://doc.rust-lang.org/std/cell/struct.RefCell.html) wrapper allows you to modify the stored value via a reference.

```rust
use std::cell::{RefCell, RefMut};

fn main() {
    let ref_cell = RefCell::new(1);
    {
        // Obtain a "mutable reference": RefMut implements DerefMut
        let mut mut_ref: RefMut<'_, i32> = ref_cell.borrow_mut();

        // Modify the value inside the RefCell
        *mut_ref = 5;
    }
    println!("{:?}", ref_cell);
}
```

`RefCell` does not break the reference safety rules; it simply moves the checks from compile-time to runtime. This means attempting to acquire both a mutable and an immutable reference to the contents of a `RefCell` simultaneously will cause a panic.

```rust
use std::cell::{RefCell, RefMut, Ref};

fn main() {
    let ref_cell = RefCell::new(1);

    let immut_ref: Ref<'_, i32> = ref_cell.borrow(); // borrowing immutable

    let mut mut_ref: RefMut<'_, i32> = ref_cell.borrow_mut();
    *mut_ref = 5;                   // ^^^ already borrowed: BorrowMutError
    
    println!("{:?}", ref_cell);
}
```

---

Now we can rewrite our singly linked list using `RefCell`:

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
enum List<T> {
    Elem(Rc<RefCell<T>>, Rc<List<T>>),
    Nil,
}

use List::*;

fn main() {
    let v = Rc::new(RefCell::new(1));

    let a = Rc::new(Elem(Rc::clone(&v), Rc::new(Nil)));

    let b = Elem(Rc::new(RefCell::new(2)), Rc::clone(&a));
    let c = Elem(Rc::new(RefCell::new(3)), Rc::clone(&a));

    *v.borrow_mut() += 10;
    println!("a after = {:?}", a);
    // Elem(RefCell { value: 11 }, Nil)
    
    println!("b after = {:?}", b);
    // Elem(RefCell { value: 2 }, Elem(RefCell { value: 11 }, Nil))
    
    println!("c after = {:?}", c);
    // Elem(RefCell { value: 3 }, Elem(RefCell { value: 11 }, Nil))
}
```

## Arc

`Rc` allows shared ownership of an object, but `Rc` is not a thread-safe type—meaning it doesn't allow shared ownership across different threads. For multi-threaded environments, there is a thread-safe version: [Arc](https://doc.rust-lang.org/std/sync/struct.Arc.html) (Atomically Reference Counted).

We will explore multi-threaded programming in Rust in the [Multithreading](../dive-deeper/multithreading.md) chapter, so for now, just remember that `Arc` is used instead of `Rc` in multi-threaded contexts.


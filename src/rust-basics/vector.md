# Vector

As we already know, the size of an array must be known at compile time, which makes it useless for scenarios where the size is calculated during the program's execution.

The [Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html) type (vector) represents a contiguous sequence of elements whose size can be determined and changed during runtime.

> [!NOTE]
> `Vec<T>` is a generic structure. We will study both generics and structures a few chapters later; however, the vector is such an omnipresent data structure that studying even basic Rust constructs without it would be very difficult. Therefore, at this stage, we will only focus on how to work with it and how it is laid out in memory.
If you are familiar with C++, you have likely already drawn an analogy with the template class `std::vector`, and you are absolutely right.
If you are familiar with Java, consider the vector a close relative of the `ArrayList<T>` class.

First, let's look at an example of using a vector:

```rust
fn main() {
  // Создаём пустой вектор
  let mut my_vec: Vec<i32> = Vec::new();

  my_vec.push(1); // Add 1 to the end of the vector
  my_vec.push(2); // Add 2 to the end of the vector
  my_vec.push(3); // Add 3 to the end of the vector

  // Copy the value of the element at index 2 (zero-based indexing) into the variable 'third'
  let third: i32 = my_vec[2];
  println!("3-rd element: {}", third);
}
```

As you can see, from a usage perspective, a vector can be thought of simply as a dynamically resizable array.

## Memory Layout

Now let's talk about how a vector is stored in memory. When we create a vector variable, only its "metadata" is placed on the stack, while the actual data is stored on the heap.

The following three fields are stored on the stack:

* A pointer to the start of the buffer in the heap — this buffer holds the actual elements of the vector.
* A length counter (len) for the number of elements currently written in the heap buffer.
* The capacity (size) of the buffer in the heap.

The layout of the vector from the example above in RAM looks like this:

![](img/vector_in_memory.svg)

The size of the heap buffer that a vector initially creates is not standardized, but in this diagram, we assumed it is equal to 5.

If the desired heap buffer size is known in advance, it can be specified explicitly by replacing `Vec::new()` with `Vec::with_capacity(size)`. This causes the vector to allocate an initial heap buffer of exactly the right size to accommodate the specified number of elements.

As elements are added to the vector, the element counter (`len`) increases. When `len` becomes equal to `capacity` — meaning the heap buffer is completely full — the vector allocates a new, larger buffer, copies all elements from the old buffer into it, and then deletes the old buffer. Subsequent elements are then added to the new buffer.

```rust
  let mut my_vec: Vec<i32> = Vec::with_capacity(3);

  my_vec.push(1);
  my_vec.push(2);
  my_vec.push(3);

  // <- At this point, the buffer with a capacity of 3 elements is full

  // Adding a 4th element will trigger the allocation of a new buffer
  // and the copying of 1, 2, and 3 into it.
  // After that, 4 will be added to the new buffer.
  my_vec.push(4);
```


### The vec! macro

In the previous example, to create a vector with elements, we first created an empty vector and then added all the necessary elements one by one. You'll agree that adding elements individually is quite inconvenient.
Therefore, since the vector is the most frequently used data structure, the Rust standard library includes a special macro `vec![]`, which handles the burden of adding elements one by one.

Using this macro, we can rewrite the example above like this:

```rust
fn main() {
  let mut my_vec = vec![1,2,3];

  let third: i32 = my_vec[2];
  println!("3-rd element: {}", third);
}
```

How this macro works will become clear only after reading the [Declarative Macros](declarative-macro.md) chapter.

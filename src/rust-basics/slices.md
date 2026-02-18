# Slices

Just as a reference points to data belonging to a specific variable, a **slice** is a special kind of reference that points to a continuous sequence of elements, usually belonging to another variable (a vector, an array, etc.).

Similarly to how a reference is created using the `&` operator, a slice is created using the `&[]` operator.

```rust
let arr: [i32; 5]  = [0, 1, 2, 3, 4]; // array
let slice1: &[i32] = &arr[..];        // slice of all array elements
let slice2: &[i32] = &arr[2..=4];     // slice of elements from index 2 to 4 inclusive

// alternatively, you could write:
// let slice2: &[i32] = &arr[2..5]; // from index 2 to index 5 exclusive
```

A slice allows you to access elements by their index (relative to the start of the slice, not the start of the original sequence).

```rust
fn main() {
  let arr = [0, 1, 2, 3, 4];
  let slice: &[i32] = &arr[2..=4]; // slice of elements from index 2 to 4
  println!("{}", slice.len()); // 3 (slice length)
  println!("{}", slice[2]);    // 4 (the element at index 2 of the slice)
}
```

Unlike regular references, which are typically ephemeral (not represented by separate cells in memory), slices are stored in memory as a pair of fields:

* The address of the first element of the sequence
* The number of elements

![](img/slice_to_array.svg)

By default, slices—just like references—are immutable. To create a mutable slice, you must use the `mut` keyword.

Let's look at an example of modifying vector elements through a mutable slice:

```rust
fn main() {
  let mut v: Vec<i32> = Vec::with_capacity(5);
  v.push(0);
  v.push(1);
  v.push(2);
  v.push(3);

  let slice: &mut [i32] = &mut v[1..3]; // slice of elements from index 1 to 3 (exclusive)
  slice[0] = 9;

  println!("v[1]: {}", v[1]); // 9
}
```

In memory, the relationship between the vector and the slice looks like this:

![](img/slice_to_vec.svg)


# Arrays

Arrays in Rust represent a contiguous sequence of values whose size is known at <ins>compile time</ins>.

The array type consists of two parts: the element type and the number of elements.

```
[element_type; number_of_elements]
```

For example, an array of three elements of type `i32`:

```rust
let arr: [i32; 3] = [1, 2, 3];
```

The size of an array cannot be determined at runtime (for example, by reading the desired size from the console and constructing it). It must be known at compile time, which makes arrays useful only in relatively rare cases.

For example, an array of four bytes is well suited for storing an IPv4 address.

```rust
fn main() {
  let mut arr: [u8; 4] = [192, 168, 0, 1];
  println!("Array is {arr:?}");
  // Prints: Array is [192, 168, 0, 1]
}
```

> [!NOTE]
> As we can see, here we use the `{:?}` formatting specifier instead of `{}` to print to the console. As mentioned in the [Printing to the Console](console-output.md) section, this is necessary because arrays do not implement the `std::fmt::Display` trait, but like most standard types, they implement `std::fmt::Debug`.

Array indexing starts at zero.

```rust
fn main() {
  let arr: [u8; 4] = [192, 168, 0, 1];
  println!("{}", arr[0]); // 192
}
```

Like any other variables, an array variable is immutable by default. To make an array mutable, it must be declared with the `mut` keyword.

```rust
fn main() {
  let mut arr = [1,2,3];
  arr[1] = 55;
  println!("Array is {arr:?}");
  // Prints: Array is [1, 55, 3]
}
```

Arrays declared inside functions are always allocated on the stack.

There is a mechanism that allows moving an array to the heap, however, finding a practical use case for such an operation is quite difficult.


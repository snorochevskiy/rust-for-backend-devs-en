# Tuples

As we know, an array in Rust is a sequence of a pre-determined size containing elements of the same type. Arrays are well-suited for storing entities like spatial coordinates: `[x, y, z]` — where all three components share the same type. But what if storing an entity requires a sequence of elements of different types?

For instance, if we are developing a filing system for an HR department, we might need to store each employee's full name, birth year, and a flag indicating whether the employee is active. These are data of different types: `String`, `u32`, and `bool`. Tuples can help solve this problem.

A **tuple** is a fixed-size sequence that can contain elements of different types.

> [!TIP]
> Java programmers might draw an analogy with the `Pair<T1,T2>` and `Triplet<T1,T2,T3>` classes from the Apache Commons library.

The syntax for declaring a tuple looks like this:

```
(value_1, value_2, ..., value_N)
```

Example: a tuple for storing an employee's name, birth year, and active status.

```rust
let employee: (&str, i32, bool) = ("John Doe", 1980, true);
```

Accessing tuple elements is done using a `.` followed by the element index (starting from zero).

```rust
println!(
  "Name: {}, birth year: {}, active: {}",
  employee.0, employee.1, employee.2
);
```

There is also a convenient syntax for tuples that allows "unpacking" the entire tuple into elements and assigning those elements to variables at once.

```rust
let (name, birth_year, is_active) = employee;
```

> [!NOTE]
This "unpacking" operation is called **destructuring assignment**. We will discuss it separately in the chapter on [Destructuring](destructuring-assignment.md)

## Returning a Tuple from a Function

One convenient use for a tuple is returning multiple values from a function.

For example, suppose we need a function that takes a sequence of numbers as an argument and splits it into two parts: the first containing all odd numbers, and the second containing all even numbers.

```rust
fn split_to_odd_and_even(numbers: &[i32]) -> (Vec<i32>, Vec<i32>) {
  let mut odds = Vec::new();  // for odd numbers
  let mut evens = Vec::new(); // for even numbers
  for n in numbers {
    if n % 2 != 0 {
      odds.push(*n);
    } else {
      evens.push(*n);
    }
  }
  (odds, evens)
}

fn main() {
  let numbers = vec![1,2,3,4,5,6,7,8,9];
  let (odds, evens) = split_to_odd_and_even(&numbers); // obtain a slice of the vector
  println!("Odd numbers:  {odds:?}");
  println!("Even numbers: {evens:?}");
}
```

As we can see, in the body of the `split_to_odd_and_even` function, we created two vectors and returned both by "wrapping" them in a tuple.

Also, note that the `split_to_odd_and_even` function expects a <ins>slice</ins> as an argument. By using the `&` operator, we create a slice pointing to the elements of the vector. Passing a sequence to a function via a slice is a common practice in Rust. This allows the function to be called for both vectors and arrays.

For instance, here is how the version using an array would look:

```rust,noplayground
fn main() {
  let numbers = [1,2,3,4,5,6,7,8,9];
  let (odds, evens) = split_to_odd_and_even(&numbers); // obtain a slice of the array
  println!("Odd numbers: {odds:?}");
  println!("Even numbers: {evens:?}");
}
```


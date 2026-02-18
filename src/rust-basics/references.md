# References

A reference is a variable that "refers" to data stored in another variable.

To create a reference to a value stored in a variable, the `&` operator is used:

```rust,noplayground
let variable: Type = value;
let reference: &Type = &variable;
```

In this case, the type of the reference will be `&original_variable_type`.

For example:

```rust
fn main() {
  let a: i32 = 5;
  let ref_a: &i32 = &a;
  println!("Value in a is {}", *ref_a); // Value in a is 5
}
```

The `*` operator in the expression `*ref_a` is used to get the value that the reference points to.

References, like variables, are immutable by default, meaning they can be used to read the value of the original variable but not to change it. To create a mutable reference, you must use `&mut` instead of `&`. Naturally, a mutable reference can only be obtained for a mutable variable:

```rust
fn main() {
  let mut a: i32 = 5;
  let ref_a: &mut i32 = &mut a;
  *ref_a = 99;
  println!("Value in a is {}", *ref_a); // Value in a is 99
}
```

> [!NOTE]
> Unlike pointers in C, which are a physical data type (i.e., a memory cell that stores an address), a reference in Rust is more of a semantic entity. This means that in most cases, creating a reference in code does not result in the creation of additional entities in the program's memory: the compiler simply replaces interaction with the reference with interaction with the value directly. At the program code level, however, the reference behaves as if it were a physical "pointer" to data stored in another variable.

In Rust, references are safe: the compiler tracks the lifetime of references relative to the lifetime of the variables whose data they point to. If you attempt to reference data belonging to a variable that has already been destroyed, the compiler will issue an error.

We will talk more about references in the [Ownership](ownership.md) chapter, as well as in the [Lifetimes](lifetimes.md) chapter.


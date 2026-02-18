# Primitive Data Types

The Rust language provides following primitive data types.

## Integer Types

| Size                                            | Signed   | Unsigned |
| ----------------------------------------------- | -------- | ----------- |
| 8 bit                                           | `i8`     | `u8`        |
| 16 bit                                          | `i16`    | `u16`       |
| 32 bit                                          | `i32`    | `u32`       |
| 64 bit                                          | `i64`    | `u64`       |
| 128 bit                                         | `i128`   | `u128`      |
| Platform-dependent<br/> (equal to pointer size) | `isize`  | `usize`     |

The default integer number type (used for variable without explicitly specified type) is `i32`. The default floating-point number type is  `f64`.

```rust
let a = 5;   // i32
let b = 5.0; // f64
```

The type of a number can be specified either explicitly or by using a suffix on the initialization value:

```rust
let a: u8 = 5;
let b = 5u8;

let c: f32 = 5.0;
let d = 5.0f32;

let e: u128 = 1;
let f = 1u128;
```

## Floating-Point Numbers

Rust provides two types for representing floating-point numbers: `f32` and `f64`. Their sizes are 32 and 64 bits, respectively.

Both types implement the [IEEE-754 standard](https://en.wikipedia.org/wiki/IEEE_754), meaning that `f32` and `f64` values can store real numbers as well as "infinity" and "not a number" (NaN).

```rust
let a: f32 = -1.0; // 1.0
let b = 5.0f32;    // 5.0
let c = a + b;     // 4.0
let d = 1.0 / 0.0; // inf
let e = a.sqrt();  // NaN
```

## bool

The boolean type in Rust is exactly what you would expect: it can store either `true` or `false`.

```rust
let a: bool = true;
let b = false;
```

A `bool` value occupies 1 byte in RAM.

## Characters

The `char` type is used to store individual text characters. In practice, it is a 4-byte number that stores the character's Unicode code.

You can specify a character in the source code as a literal symbol enclosed in single quotes.

```rust
let a = 'a';
let smile = '☺';
let book = '本';
```

## Unit

The Unit type is an analogue of the `void` type in C, Java, C#, etc. It is typically used to specify the return type of functions that do not return a specific value.

Despite the type is called "Unit," it is represented in code as `()`.

The fundamental difference between the Unit type (which is a singleton set) and the `void` type (which represents an empty set) is that the Unit type has exactly one value: `()`. We can even create a variable of the Unit type and assign `()` to it.

```rust
let a: () = ();
```

> [!TIP]
> What's wrong with the void type?
> 
> The Unit type is common in languages influenced by functional programming. In functional programming, a function is primarily viewed as a mathematical function — i.e., a <ins>mapping</ins> from a set of arguments to a set of results. This is where the fundamental problem with the `void` type arises: in math the empty set that contains no values. This complicates the modeling of the types system for such operations like composition of functions.
> 
> In the pure functional language Haskell, there is even a special function called `absurd` that accepts an argument of type `Void` and returns some value.
> 
> ```haskell
> absurd :: Void -> a
> ```
> 
> What this value is and what its type is — doesn't matter, as it is impossible to call this function: to call it, you would need an element from the empty set `Void`, which does not exist.


## Never type

Another interesting type, that came from functional programming, is the never type.

In code, this type is denoted by `!`.

This type is almost never explicitly specified and is rarely even observed in code. Furthermore, if you don't know it exists, you can write Rust for years without ever suspecting its presence.

At this stage, we won't be able to fully dive into this type. We will only say that the compiler uses this type for expressions that never return a value to the calling code. For example, a function that exits the program.

We will look at the never type in more detail in the [Functions](functions.md) chapter.

## Type Casting

The `as` operator is used to convert one data type into another.

Syntax:

```
value as Type
```

Example:

```rust
fn main() {
    let a: i32 = 5;
    let b: i64 = a as i64;
    let c: i32 = b as i32;

    let d: f32 = 7.0;
    let e: i32 = d as i32;

    let f: bool = true;
    let g: i32 = f as i32; // 1
    let h = false as i32;  // 0

    let i = 'A' as i32; // 65
    let j = 66 as char; // B
}
```

Rust is a strictly typed language, so it lacks the implicit type conversions like in C. Any type conversion must be explicitly specified using the `as` operator.

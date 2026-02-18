# Printing to the Console

As we have already seen in the "Hello World" example from the [First Look](first-look.md) chapter, the `println!` macro is used to print to the console.

The `println!` macro is not as simple as it seems. In fact, it isn't a function at all — it’s a macro. If you are familiar with macros in C, let me reassure you right away: macros in Rust are much safer and far more convenient.

We will dive into how Rust macros are structured and exactly how `println!` works later in the [Declarative Macros](declarative-macro.md) chapter. For now, let’s simply look at some usage examples that we will need for the upcoming material.

***

To print a string to the console, simply pass it as an argument to the `println!` call.

```rust
println!("Print just text");
```

If you need to print the value of a variable, you must pass two arguments to `println!`:

* 1st argument: A format string containing the `{}` placeholder where you want the variable's value to appear.
* 2nd argument: The variable whose value will replace the `{}` placeholder.

For example:

```rust
fn main() {
    let magic_number: i32 = 5;
    println!("Number is {}", magic_number);
}
```

If you compile and run this program, you will see the following:

```
$ rustc main.rs
$ ./main
Number is 5
```

> [!NOTE]
> This is an example of building and running on Linux. On Windows, it looks like this:
>
> ```
> rustc main.rs
> main.exe
> Number is 5
> ```

If you want to print two variables, you must include two `{}` placeholders in the format string and then pass both variables:

```rust
let number_1 = 5;
let number_2 = 6;
println!("Number 1 is {}, number 2 is {}", number_1, number_2);
```

Alternatively, the variable can be placed directly inside the curly braces rather than after the format string:

```rust
let magic_number: i32 = 5;
println!("Number is {magic_number}");
```

There is also a combined approach: defining an alias inside {} and then binding a value to that alias afterward.

```rust
let magic_number: i32 = 5;
println!(
    "Number is {num}, num in the power of two is {square}",
    num = magic_number,
    square = magic_number * magic_number
);
```

***

Only values of types that implement the `std::fmt::Display` trait can be printed this way. We will discuss traits later; for now, all you need to know is that `{}` can only be used with types where the method for converting them into a string is explicitly defined.

In the Rust standard library, all primitive types already implement `std::fmt::Display`, allowing them to be printed directly via `println!`. However, this is not the case for most complex types (such as arrays).

That said, if you replace `{}` with `{:?}` in the format string, you can print values of types that implement the `std::fmt::Debug` trait — which covers the vast majority of types.

Don't worry too much about this detail for now; we will revisit it many times.

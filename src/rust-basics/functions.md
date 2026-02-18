# Functions

Like in other programming languages, functions in Rust are a mechanism that allows you to break a program into separate, granular subroutines.

Function declaration syntax:

```rust,noplayground
fn func_name(arg1: Type1, arg2: Type2) -> ReturnType {
    // function body
}
```

Example:

```rust
fn sum(a: i32, b: i32) -> i32 {
    a + b
}

fn safe_divide(a: f32, b: f32) -> f32 {
    if b != 0.0 {
        a / b
    } else {
        0.0
    }
}

fn main() {
    let a = sum(1, 2);
    println!("{a}");
    
    let b = safe_divide(12.0, 4.0);
    println!("{b}");
}
```

When declaring function arguments, you can include a trailing comma after the last argument, and the behavior will be the same as without it.

```rust,noplayground
fn sum(a: i32, b: i32,) -> i32 { .. }
```

## return

As we can see, the last evaluated value is automatically the result of the function.

If you need to exit a function "early," you should explicitly use the `return` operator:

```rust
fn safe_divide(a: f32, b: f32) -> f32 {
    if b == 0.0 {
        return 0.0;
    }
    a / b
}

fn main() {
    println!("12 / 3 = {}", safe_divide(12.0, 3.0));

    println!("12 / 0 = {}", safe_divide(12.0, 0.0));
}
```

## Nested functions

Rust allows you to declare a function inside another function.

If a part of a function's logic is well-granulated and used several times but isn't needed anywhere else in the program, it can be moved into a separate internal function.

```rust
// Returns the i-th element of the Fibonacci sequence.
// Sequence indexing starts from zero.
fn fibonacci_nth_element(index: usize) -> u32 {
    if index == 0 {
        return 0;
    }
    if index == 1 {
        return 1;
    }
    // Calculates the i-th element of the Fibonacci sequence
    // * x0            - the i-th element
    // * x1            - the (i+1)-th element
    // * next_index    - the index of the next (i+2) element
    // * desired_index - the index of the element we are looking for
    fn next_fibonacci(x0: u32, x1: u32, next_index: usize, desired_index: usize) -> u32 {
        let x2 = x0 + x1;
        if next_index == desired_index {
            x2
        } else {
            next_fibonacci(x1, x2, next_index + 1, desired_index)
        }
    }

    next_fibonacci(0, 1, 2, index)
}

fn main() {
    println!("{}", fibonacci_nth_element(0)); // 0
    println!("{}", fibonacci_nth_element(1)); // 1
    println!("{}", fibonacci_nth_element(2)); // 1
    println!("{}", fibonacci_nth_element(3)); // 2
    println!("{}", fibonacci_nth_element(4)); // 3
}
```

## return и never type

> [!TIP]
> The information below is not strictly necessary for general Rust programming, but rather provides a better understanding of the type system.

In the [Primitive Data Types](primitive-types.md) chapter, we mentioned the never type `!`, which is used for expressions that do not return control to the calling code.

The `return` operator returns the `!` type because it terminates the function's execution and, therefore, returns nothing back into the function's flow.

```rust
fn gen_num() -> i32 {
    // Variable v has the type !
    let v = return 5;
}
  
fn main() {
    let a = gen_num();
}
```

The never type doesn't represent actual data; it acts as a placeholder to "glue" the Rust type system together. Let’s look at an example:

```rust
fn safe_divide(a: f32, b: f32) -> f32 {
    let non_zero_divider: f32 =
        if b != 0.0 {
            b
        } else {
            return 0.0
        };
    a / non_zero_divider
}

fn main() {
    println!("12 / 0 = {}", safe_divide(12.0, 0.0));
}
```

Notice that the type of the `non_zero_divider` variable is `f32`. How is this possible? The result type of the first branch of the `if` expression is `f32`, but the result type of the second branch is `!` (never type).

The reason is that the never type can be automatically coerced into any other type. This is perfectly safe because, in reality, the conversion of a `!` type value into another type never actually happens (as control flow has already exited). This unique property of the never type is the "glue" that allows types to align in such expressions.

> [!TIP]
> Programmers familiar with Scala might find an analogy with the `Nothing` type.


## const functions { #const-functions}

In Rust, you can define a function that can be executed at compile time. Such a function is marked with the `const` keyword.

```rust
const fn func_name(arg1: Type1, arg2: Type2) -> ReturnType {
    // function body
}
```

For example:

```rust
const PI: f32 = 3.14;
const TAU: f32 = double(PI);

const fn double(num: f32) -> f32 {
    num * 2.0
}

fn main() {
    println!("Tau = {TAU}");
}
```

## static variables { #static-variables }

A function can contain [static variables](variables.md#static), whose values are preserved between function calls.

This is best demonstrated with an example:

```rust
fn sum_with_previous(x: i32) -> i32 {
    static mut PREV: i32 = 0; // static variable
    unsafe {
        let result = PREV + x;
        PREV = x;
        result
    }
}

fn main() {
    println!("{}", sum_with_previous(1));  // 1
    println!("{}", sum_with_previous(2));  // 3
    println!("{}", sum_with_previous(7));  // 9
    println!("{}", sum_with_previous(-6)); // 1
}
```

The `sum_with_previous` function adds the argument's value to the argument value from its previous call. To achieve this, it uses the static variable `PREV`, which lives outside the function call's context.

As you can see, `PREV` is initialized to zero. This initialization is only performed for the very first call to the function.

It is also important to note that all interaction with a mutable static variable must occur within an `unsafe` block. This is because the function could potentially be called from multiple parallel threads simultaneously, which could lead to one thread reading or writing an incorrect value of `PREV`. This is a classic example of a data race, which is unacceptable in safe Rust. This is why the `unsafe` block is required.

This is the first time we've encountered an `unsafe` block. We will explore it in more detail later, but for now, understand that the author of the code takes full responsibility for the safety of the code within an `unsafe` block. Therefore, you should avoid using unsafe Rust unless absolutely necessary.

As you might have guessed, mutable static variables are inherently unsafe and can generally only be used without additional synchronization mechanisms in a single-threaded environment. Fortunately, in back-end applications, you will likely never use them — we've covered them here simply to provide a deeper understanding of how functions work.

# Variables

## Declaring a Variable

The syntax for declaring variables in Rust looks like this:

```rust,noplayground
let variable_name: type = value;
```

For example:

```rust
let a: i32 = 5;
```

If a value is assigned to a variable immediately upon declaration, the type can be omitted because the compiler will infer it automatically:

```rust
let a = 5;
```

A variable can also be declared first and assigned a value later. In this case, explicitly specifying the type is also optional, as the compiler can match the declaration with the subsequent initialization.

```rust
let a;
a = 5;
```

## Мутабельность

> [!TIP]
> Mutability refers to the ability of a variable to be changed (from the word "mutable").

By default, variables in Rust are immutable. This means that once a value has been assigned to a variable, the compiler will not allow you to write a different value into it.

The following program:

```rust,compile_fail
fn main() {
    let a = 1;
    a = 5;
}
```

will cause a compilation error:

```
error[E0384]: cannot assign twice to immutable variable `a`
 --> main.rs:3:5
  |
2 |     let a = 1;
  |         - first assignment to `a`
3 |     a = 5;
  |     ^^^^^ cannot assign twice to immutable variable
```

To make a variable mutable, you must declare it using the `mut` keyword.

```rust
let mut a = 1;
a = 5;
```

## Variable Naming

Variable names can contain only letters (including valid Unicode letters), numbers, and underscores. However, a variable name cannot start with a digit. Rust keywords cannot be used as variable names.

```rust
let a = 1;
let b_x_5 = 2;
let this_is_a_variable = 3;
let 変数名 = 4;
```

In Rust, it is standard practice to use snake_case for variable names:

* Variable names should start with a lowercase letter.
* If a name consists of multiple words, they should be separated by an underscore.

For example:

```rust
let number = 1;
let some_number = 2;
let coordinate_2d_x = 2.11;
```

If you need to use a keyword as a variable name (which might be necessary, for example, when deserializing a field from a binary format), you must add the `r#` prefix to the variable name:

```rust
let r#if = 5;
```

## Constants

In addition to variables, Rust features constants. While variables are typically stored on the stack, constants are generally stored in the code segment.

The syntax for declaring a constant is:

```rust
const NAME: Type = value;
```

Unlike variables, you <ins>must always</ins> explicitly specify the data type for constants.

Constant names should consist of uppercase letters, using underscores as separators.

```rust
const PI: f32 = 3.14;
const ANONYMOUS_NAME: &str = "anonymous";
```

## static

Static variables are created at the very start of the program and exist until it finishes. However, unlike constants, static variables:

* Are initialized at runtime, not during compilation.
* Can be mutable.
* Can contain objects that store data on the heap.

A static variable is declared using the `static` keyword:

```rust
static NAME: Type = value;
```

Static variables can be either global or local to a function. We will cover both types in detail in their respective chapters.

A mutable static variable is a potentially unsafe resource because it can be modified by multiple threads simultaneously. Therefore, working with mutable static data is an `unsafe` operation, which we will discuss later.

> [!NOTE]
> While constants are usually stored in the code segment, static variables are stored in the `.bss` (Block Started by Symbol) segment or the data segment.

## The _ Prefix

The compiler issues warnings for every unused variable—a variable that is declared and perhaps initialized but never used in any expression.

If we intentionally want to keep such a variable in the code (for example, as a placeholder for future functionality), we can prefix the variable name with an underscore.

For example:

```rust
let _some_unused_variable = 5;
```

The compiler will not issue warnings about this variable being unused.

## "Discarded" Variables { #discarded-variable }

If a variable consists solely of a single underscore `_`, it is a "discarded" variable. Such a variable semantically exists, but it cannot be accessed at all.

```rust
let _ = 5;
```

One common scenario for using discarded variables is intentionally ignoring the result of a function. We will explore this example in the [Result](result.md) chapter.

We will also see these variables in the [Destructuring](destructuring-assignment.md) and [Pattern Matching](pattern-matching.md) chapters.

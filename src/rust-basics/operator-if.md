# if operator

Just like in most programming languages, Rust features the `if` conditional operator.

Its syntax looks like this:

```rust,noplayground
if condition_1 {
    expression_1
} else if condition_2 {
    expression_2
} else {
    expression_3
}
```

For example:

```rust
fn main() {
    let a = 10;

    if a % 2 == 0 {
        println!("a is even");
    } else {
        println!("a is odd");
    }
}
```

In Rust, unlike C and Java, the body of a conditional branch <ins>must</ins> be enclosed in curly braces, even if the branch contains only a single expression.

## Результат оператора if

The `if` operator is an **expression**, meaning it returns a value that can be assigned to a variable. The result of the entire `if` expression is the result of the last expression in the branch that was executed.

```rust
let a = -5;
let mod_a: i32 =
    if a < 0 {
        -a
    } else {
        a
    };
println!("{mod_a}"); // 5
```

It is important to note here that for the `if` operator to return a value, the last expression in the conditional branch <ins>must not</ins> end with a semicolon (`;`). This is because the body of a conditional branch is always a scope. As we already know from the [Scopes](scopes.md) chapter, if the last expression in a scope ends with a `;`, the scope returns `()`.

```rust
let a = -5;
let mod_a: () =
    if a < 0 {
        -a; // same as { -a; () }
    } else {
        a;  // same as { a; () }
    };
println!("{mod_a:?}"); // Prints: ()
```


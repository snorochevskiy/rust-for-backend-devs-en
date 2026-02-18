# Scopes

In Rust, a variable's lifetime is limited by its scope — the block of curly braces in which it is declared. When a scope ends, all variables declared within it are automatically dropped.

```rust
fn main() {
  let a = 1;

  {
    let b = 2;
    // At this point, both a and b exist
  } // Scope ends: b is dropped

  // At this point, only a exists

  let c = 3;
} // Scope ends: a and c are dropped
```

## The Value of a Scope

In Rust, every scope returns a value. This means the result of a scope can be assigned to a variable. The value returned by the scope is the last expression evaluated within it.

```rust
let a: i32 = {
  let x = 1;
  let y = 2;
  x + y // last evaluated expression
};
println!("{a}"); // 3
```

Be careful: for the result of the last expression to become the result of the scope, there <ins>must not</ins> be a semicolon after it.

The reason is that, unlike C, where `;` marks the end of the <ins>current</ins> expression, in Rust, `;` serves as a separator <ins>between</ins> expressions. This means, for example, the expression:

`{ a; }`

is interpreted as:

`{ a; () }`

Consequently, the result of the entire scope will be the <ins>Unit</ins> type, not `a`.

Here is what the previous example would look like if a semicolon were mistakenly placed at the end of the scope:

```rust
let a: () = {
    let x = 1;
    let y = 2;
    x + y;
};
```

The fact that a scope returns a value is utilized in several other constructs that we will discuss later.

# Lifetimes

By now, we know that in Rust, every object has exactly one owner, but that owner can lend the object via references to other parts of the code. In doing so, the compiler verifies that the **lifetime** of the scope that borrowed the object does not exceed the lifetime of the object's owner.

Now let’s look at the following code:

```rust
// This function takes two string slices and returns the one
// that points to the longer string.
fn take_longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let l = take_longest("aaa", "bbbb");
}
```

An attempt to compile this code will result in the error:

> error\[E0106]: missing lifetime specifier\
> help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from \`x\` or \`y\`\
> help: consider introducing a named lifetime parameter

This error indicates that the compiler is unsure how the lifetime of owner `x`, the lifetime of owner `y`, and the lifetime of the variable that receives the function's result relate to one another.

To make this clearer, let’s look at a specific way the `take_longest` function might be used:

```rust,noplayground
let s1 = String::from("aaa");
let longest;
{
    let s2 = String::from("bbbb");
    longest = take_longest(s1.as_str(), s2.as_str());
}
```

In this scenario, a reference to a string owned by `s2` is stored in the variable `longest`. The problem is that `longest` belongs to a scope that "lives" longer than the scope containing `s2`. As we mentioned before, the compiler ensures that a reference never outlives the variable that owns the underlying object.

To solve this, we must explicitly define the relationship between lifetimes of the objects referenced by the function arguments and the lifetime of the target variable that will accept the result of the function.

These relationships are defined using **lifetimes**.

```rust,noplayground
fn take_longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

In the function header `take_longest<'a>`, we declare a named lifetime `'a`. Then, for each reference, we specify which lifetime it belongs to by placing the name after the `&` symbol.

The lifetime annotations in the `take_longest` signature can be read like this:

> Given some lifetime `'a` of arbitrary length. The owners of objects referenced by `x` and `y` belong to the same scope that is covered by lifetime `'a`. Furthermore, the lifetime of the variable receiving the function's result must not exceed the duration of `'a`.

After we have defined the lifetimes, the following attempt to use `take_longest` will result in a compilation error:

```rust,compile_fail
fn main() {
  let s1 = String::from("aaa");
  let longest;
  {
    let s2 = String::from("bbbb");
    longest = take_longest(s1.as_str(), s2.as_str()); // does not live long enough
  }
  println!("The longest string is {}", longest);
}
```

## The `'static` Lifetime

In Rust, there is one "global" lifetime: `'static`. A reference with a `'static` lifetime means the object is guaranteed to live from the current moment until the end of the program's execution.

For example, a constant reference to a string literal always has a `'static` lifetime.

```rust
const text: &'static str = "some string";

fn main() {
    println!("{text}");
}
```

## The Complexity of Lifetimes

Lifetimes are often considered one of the most difficult concepts for those whole learn Rust. If you find yourself struggling with lifetimes, don't overcomplicate things — in most situations it is fine to make an owned copy of the object instead of dealing with complex reference relationships.

As you become more comfortable with the language, you can dive deeper into the nuances of lifetimes.


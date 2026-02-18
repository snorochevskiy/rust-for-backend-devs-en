# Pattern Matching

In addition to the `if` statement, Rust features another branching operator that we haven't covered yet — the `match` operator, often referred to as **pattern matching**.

This operator is a hybrid of the classic switch-case (from C or Java) and destructuring assignment.

Syntax:

```rust,noplayground
match value {
    pattern 1 => expression 1,
    …
    pattern N => expression N,
}
```

The operator sequentially compares the patterns against the value: if a pattern matches, the corresponding expression is executed and the `match` block completes its execution. Otherwise, it proceeds to check the next pattern in the list.

A pattern can be:

* A constant
* A reference
* A variable or a variable with a conditional block (guard)
* A destructuring assignment pattern

Let’s look at these options in order of increasing complexity.

## match as switch-case

The simplest way to use the `match` operator is for direct value comparison, similar to switch-case in C or Java.

As an example, let's check if the value of a numeric variable is `0` or `1`:

```rust
fn main() {
    let a = 1;
    match a {
        0 => println!("It is 0"),
        1 => println!("It is 1"),
        _ => println!("Neither 0 nor 1"),
    }
    // Prints: It is 1
}
```

The compiler ensures that the `match` arms cover <ins>all</ins> possible variants of the variable's value.
This is why, after the arms for `0` and `1`, we included an arm with the `_` pattern. This pattern matches absolutely any value and serves as a _default_ case (if drawing an analogy with switch-case) or an _else_ branch (if comparing it to an `if` statement).

In reality, the `_` pattern is just a regular "discarded" variable, similar to those we saw in destructuring patterns. Instead of a "discarded" variable, you can specify a variable with any valid name; the value passed into `match` will then be bound to that variable.

```rust
fn main() {
    let a = 5;
    match a {
        0 => println!("The number is 0"),
        1 => println!("The number is 1"),
        x => println!("The number is {x}"),
    }
    // Prints: The number is 5
}
```

Keep in mind that a variable pattern like this matches any value, so it must be placed at the very end of the pattern list.

You can also list multiple values within a single pattern:

```rust
fn main() {
    let a: u32 = 7;
    match a {
        0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 =>
            println!("The number is less than 10"),
        _ =>
            println!("The number is equal to or greater than 10"),
    }
    // Напечатает: The number is less than 10
}
```

When checking numbers in a pattern, instead of listing all values manually, you can specify a range:

```rust
fn main() {
    let a: u32 = 44;
    match a {
        0 ..= 9 => // From 0 to 9 inclusive
            println!("The number is less than 10"),
        10 .. 100 => // From 10 to 100 exclusive
            println!("The number is in range [10,99]"),
        _ =>
            println!("The number is equal to or greater than"),
    }
    // Напечатает: The number is in range [10,99]
}
```

## Variable Binding

In the previous example, we specified a range of values. But what if we want to know not only the range into which a value falls, but also the value itself? We could, of course, directly access the variable passed into `match`, but this isn't always possible (for instance, if an arithmetic expression or a function call was passed into `match`).

In such cases, **variable binding** comes to the rescue. This is a variable that stores the value that satisfied the pattern.

The syntax is:

```
binding @ pattern => expression
```

Now, let's rewrite the previous example using bindings:

```rust
fn main() {
    match 22 + 22 {
        x@(0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9) =>
            println!("{x} is less than 10"),
        x@10 .. 100 =>
            println!("{x} is in range [10,99]"),
        x =>
            println!("{x} is equal to or greater than"),
    }
    // Will print: 44 is in range [10,99]
}
```

## match as an Expression

The `match` operator, like the `if` operator, returns a value. The result of the entire match block is the result of the branch that was executed.

Here is an example of a `match` operator being used to get the absolute value of a number (the non-negative part):

```rust
let a = -5;
let absolute = match a {
  .. 0 => -a, // range from negative infinity to 0 (exclusive)
  _ =>     a,    
};
println!("{absolute}"); // 5
```

## Comparing with String Literals

Assume we have a `String` object and we want to check its content using a `match` operator. We might try to write the check like this:

```rust
fn main() {
    let name = String::from("Robert Smith");

    let is_anonymous = match name {
        "Anonymous".to_string() => true,
        "John Doe".to_string()  => true,
        _                       => false,
    };
}
```

Compiling this program will result in an error:

```
"Anonymous".to_string() => true,
^^^^^^^^^^^^^^^^^^^^^^^ not a pattern
```

The issue is that, as we mentioned at the beginning, a pattern can be a constant, a reference, a variable, or a destructuring assignment pattern. The expression `"Anonymous".to_string()` is none of the above.

> [!NOTE]
> Here, when we talk about a "variable," we mean a "pattern variable" for assignment (or destructuring assignment), not a variable declared outside the `match` block containing some value. In other words, the following code does <ins>not</ins> work the way you might expect:
> 
> ```rust
> fn main() {
>     let anonymous = "Anonymous".to_string();
>     let john_doe = "John Doe".to_string();
> 
>     let name = String::from("Robert Smith");
> 
>     let is_anonymous = match name {
>         anonymous => true, // This branch will be executed
>         john_doe  => true,
>         _         => false,
>     };
>     println!("{is_anonymous}"); // true
> }
> ```
> 
>This program prints "true" because the variable `anonymous` inside the `match` block is not perceived as the variable declared above containing the string "Anonymous". Instead, it is treated as a <ins>new</ins> pattern variabl for binding. Naturally, a single variable without additional conditions is a pattern that matches absolutely any value.

The only remaining option is to use string literals, as they are constants.

```rust
fn main() {
    let name = String::from("Robert Smith");
    let is_anonymous = match name.as_str() {
        "Anonymous" | "John Doe" => true,
        _                        => false,
    };
}
```

Note that we are not passing the `name` variable itself into the `match` operator, but rather a string slice of it — `name.as_str()`.

## match for Slices

For slices, you can use patterns that look exactly like destructuring assignment patterns for arrays.

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];

    let s = match v.as_slice() {
        []            => 0,
        [a, b, c, ..] => a + b + c,
        _             => -1,
    };

    println!("{}", s); // 6
}
```

As a reminder: slices cannot be used in "ordinary" destructuring assignments because the compiler cannot guarantee that the number of variables in the destructuring pattern matches the number of elements in the slice.

However, `match` does not have this problem. If a slice doesn't match a specific pattern, `match` simply moves on to the next pattern and attempts to match the slice against it.

## Matching on Structs

The syntax for struct patterns in pattern matching is very similar to the syntax for destructuring assignment; however, it additionally allows you to specify specific values for fields.

```rust
struct Person { name: String, age: u32 }

fn main () {
    let p = Person { name: String::from("John"), age: 17 };
    match p {
        Person { name, age: 1 .. 18 } => println!("Person {name} is not adult"),
        Person { name, age: 18 }      => println!("Person {name} just turned 18"),
        Person { name, .. }           => println!("Person {name} is adult"),
    }
    // Will print: Person John is not adult
}
```

## Match Guards

You can specify an additional condition in patterns using the `if` keyword. These are often called "match guards."

```rust
struct Person { name: String, age: u32 }

fn main () {
    let p = Person { name: String::from("John"), age: 17 };
    match p {
        Person { name, age } if age < 18 =>
            println!("Person {name} is not adult"),
        Person { name, age } if age == 18 =>
            println!("Person {name} just turned 18"),
        Person { name, .. } =>
            println!("Person {name} is adult"),
    }
    // Will print: Person John is not adult
}
```

The condition in an `if` block can:

* Be `compound`, meaning it can contain multiple conditions combined via `||` and `&&` operators.
* Access pattern fields, bindings, and any other names available within that scope.

We can use an `if` block to rewrite our string example:

```rust
fn main() {
    let name = String::from("Robert Smith");

    let is_anonymous = match name {
        s if s == "Anonymous" => true,
        s if s == "John Doe"  => true,
        _                     => false,
    };
    println!("{is_anonymous}"); // false
}
```

While this code works correctly, matching against string literals directly is generally considered more elegant.


## ref

As we know, destructuring assignment consumes (moves) the assigned object. If we don't want to destroy the object but simply want to obtain a reference to its field, we can "destructure" the object by reference rather than by value:

```rust
#[derive(Debug)]
struct Person { name: String, age: u32 }

fn main() {
    let mut person = Person { name: String::from("Anonymous"), age: 25 };

    let Person {name, ..} = &mut person;
    *name = "John Doe".to_string();

    // The person object is still "alive"
    println!("{person:?}"); // Person { name: "John Doe", age: 25 }
}
```

Destructuring patterns in a `match` expression are no exception: they also consume the object passed to them. But can we pass an object to a `match` statement by reference, just like in "regular" destructuring assignment? Yes, we can:

```rust
#[derive(Debug)]
struct Person { name: String, age: u32 }

fn main () {
    let mut person = Person { name: String::from("Anonymous"), age: 25 };
    match &mut person {
        Person { name, .. } if name == "Anonymous" => {
            *name = "John Doe".to_string();
        },
        Person { .. } => (),
    }
    println!("{person:?}"); // Person { name: "John Doe", age: 25 }
}
```

However, specifically for pattern matching, there is the `ref` keyword. It allows you to bind an object's field by reference within a destructuring pattern, even if the object was passed to the `match` by value. If all fields are captured via `ref` (or ignored), the original object is not consumed.

```rust
#[derive(Debug)]
struct Person { name: String, age: u32 }

fn main () {
    let mut person = Person { name: String::from("Anonymous"), age: 25 };
    match person {
        Person { ref mut name, .. } if name == "Anonymous" => {
            *name = "John Doe".to_string();
        },
        Person { .. } => (),
    }
    println!("{person:?}"); // Person { name: "John Doe", age: 25 }
}
```

As you can see, we passed the `person` variable into the `match` by value here, not by reference. But because we accessed the `name` field using a `ref` binding rather than by value, the object was not moved or destroyed. Consequently, the `person` variable remains valid after the `match` block.

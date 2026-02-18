# Option

By this point, we’ve covered all the core language constructs. However, to complete our study of the language basics, we still need to examine several standard types that are tightly integrated into Rust and permeate its entire standard library.

The first one we will look at is the most frequently used type — `Option`.

## Background

When writing real-world programs, situations often arise where you need to represent the absence of a value. For example, to store a person's full name (first, last, and middle name), we might create a structure like this:

```rust
struct FullName {
    first_name: String,
    last_name: String,
    middle_name: String,
}
```

However, there are situations where a middle name might be missing, and this needs to be represented in the code somehow.

Another ubiquitous example involves I/O operations: when reading data from an external source, we can never guarantee that we will receive all expected values. For instance, we might request a record from a database by its ID, but that record may simply not exist.

Traditionally, in imperative languages of previous generations, this problem is solved in one of three ways:

Approach 1: Reserving a specific value to indicate absence. For example, using `-1` for a missing ID or an empty string for a missing middle name. This approach is very common in the C standard library and various system APIs. Its disadvantage is that, first, it's not always possible to set aside a value to serve as an "absence" indicator, and second, the API user must be aware of such a convention.

Approach 2: Introducing an additional flag. This involves a boolean field that indicates whether another field is "empty." For example:

```rust
struct FullName {
    first_name: String,
    last_name: String,
    middle_name: String,
    is_middle_name_empty: bool,
}
```

This approach avoids the need to reserve a "empty" value, but it is extremely inconvenient for writing program logic.

Approach 3: Using a null pointer as an indicator. While convenient, this is the cause of the most common error in C programs—segmentation faults (and `NullPointerException` in Java). Additionally, this approach requires placing values on the heap, which can negatively impact program performance.

## Option for "Empty" Values

To represent missing values, Rust uses the [Option](https://doc.rust-lang.org/std/option/enum.Option.html) type, which is declared as follows:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

As we can see, this is a generic enumerated type consisting of:

* A generic tuple structure `Some(T)`
* A unit structure (singleton) `None`

Let’s figure out how to work with `Option`. Suppose we want to make an `i32` variable that can be "empty."

```rust
let mut maybe_i32: Option<i32>;
maybe_i32 = Some(5); // Assigning a value to the variable
maybe_i32 = None;    // Now the variable is "empty"
```

Using `Option`, we can rewrite our full name structure example as follows:

```rust
struct FullName {
    first_name: String,
    last_name: String,
    middle_name: Option<String>,
}
```

Another example: a function that returns a record from a database by its ID might look like this:

```rust,noplayground
fn get_record_by_id(id: u64) -> Option<Record> { ... }
```

As we can see, `Option` allows storing a potentially empty value while:

* Not needing to reserve a specific value to indicate absence.
* Not introducing additional inconvenient flags, because `Option` is an enum and already contains this flag (the discriminant).
* Avoiding the fear of null pointer errors.

## Extracting Values from Option

Now let's look at how to extract a value from an `Option`.

The most straightforward way is the `unwrap` method, which works as follows:

* If the option contains a value (i.e., it is a `Some(T)` object), the `unwrap` method will return the stored value.
* Otherwise, the program will terminate with a panic.

```rust
fn main() {
    let o: Option<i32> = Some(5);
    let i: i32 = o.unwrap();
}
```

Obviously, using `unwrap` is very unsafe. Therefore, there is the `unwrap_or` method, which allows you to specify a default value in case the option is "empty."

```rust
fn main() {
    let o: Option<i32> = None;
    let i: i32 = o.unwrap_or(1); // 1
}
```

Another way to extract a value is by using the `match` operator.

```rust
fn main() {
    let o: Option<i32> = Some(5);
    let i: i32 = match o {
        Some(v) => v,
        None    => 1,
    };
}
```

Of course, we don't necessarily have to return a value from the `match` operator if our program logic doesn't require it. For example, we can simply print different outputs:

```rust
fn main() {
    let o: Option<i32> = Some(5);
    match o {
        Some(v) => println!("Number is {v}"),
        None    => println!("Number is empty"),
    };
}
```

It is also very convenient to use the `if-let` operator with `Option`.

```rust
fn main() {
    let o: Option<i32> = Some(5);
    if let Some(v) = o {
        println!("Number is {v}");
    } else {
        println!("Number is empty");
    };
}
```

## Option Combinators

At first glance, the style of working with `Option` might seem slightly cumbersome. However, `Option` has a range of convenient combinators that make working with it simpler and more expressive.

The first combinator, the [map](https://doc.rust-lang.org/std/option/enum.Option.html#method.map) method, allows you to transform the value of an option if it exists, or do nothing if the option is empty. The `map` method takes a closure (or a function pointer) as an argument and applies it to the value inside the `Option` if it is `Some`.

<pre class="ascii-diagram">
┌─────────┐                ┌─────────┐
│Option   │ .map(|x| x+1)  │Option   │
│         │         │      │         │
│┌───────┐│         V      │┌───────┐│
││Some(5)├───────> 5+1 ────>│Some(6)││
│└───────┘│                │└───────┘│
└─────────┘                └─────────┘

┌─────────┐                ┌─────────┐
│Option   │ .map(|x| x+1)  │Option   │
│         │                │         │
│┌───────┐│                │┌───────┐│
││ None  ├─────────────────>│ None  ││
│└───────┘│                │└───────┘│
└─────────┘                └─────────┘
</pre>

Example:

```rust
fn main() {
    let s1: Option<i32> = Some(5);
    let s2: Option<i32> = s1.map(|a| { a + 1 });
    println!("{s2:?}"); // Some(6)
    
    let e1: Option<i32> = None;
    let e2: Option<i32> = e1.map(|a| { a + 1 });
    println!("{e2:?}"); // None
}
```

A more real-world example: suppose there is a function that retrieves a user object from a database by ID. If the user object exists, we want to extract the value of the "name" field.

```rust,noplayground
struct User {
    id: u64,
    name: String,
}

fn get_user_by_id(id: u64) -> Option<User> {
    // Запрос в БД
}

fn get_user_name_by_id(id: u64) -> Option<String> {
    get_user_by_id(id)
        .map(|user| user.name)
}
```

***

Another combinator [flatten](https://doc.rust-lang.org/std/option/enum.Option.html#method.flatten) converts a double wrapper `Option<Option<T>>` into a single `Option<T>`.

```rust
fn main() {
    let o1: Option<i32> = Some(1);
    let o2: Option<Option<i32>> = o1.map(|a| Some(a + 1)); // Some(Some(2))
    let o3: Option<i32> = o2.flatten(); // Some(2)
}
```

This combinator can be useful when your logic involves multiple `Option` values. For example, we want to get a middle name (which might not exist) from a user object in the database (which also might not exist).

```rust,noplayground
struct User {
    id: u64,
    first_name: String,
    last_name: String,
    middle_name: Option<String>,
}

fn get_user_by_id(id: u64) -> Option<User> {
    // Database query logic
}

fn get_user_middle_name_by_id(id: u64) -> Option<String> {
    get_user_by_id(id)
        .map(|user| user.middle_name)
        .flatten()
}
```

***

The [and_then](https://doc.rust-lang.org/std/option/enum.Option.html#method.and_then) method works like a combination of `map` and `flatten`: it first applies a function to the option's content that returns an `Option`, and then "flattens" the two options into one.

```rust
fn main() {
    let o1: Option<i32> = Some(1);
    let o2: Option<i32> = o1.and_then(|a| Some(a + 1)); // Some(2)
}
```


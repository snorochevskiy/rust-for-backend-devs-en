# Enums

Enumerations, or simply "enums," in Rust can take two forms:

* A simple list of values, similar to C, C++, Java, etc.
* A container for types

Let’s examine each of these forms.

## C-style Enums

If we need an enumeration similar to those in C, the following syntax is used for declaration:

```rust
enum EnumName {
    Variant1,
    Variant2,
    …,
    VariantN
}
```

For example:

```rust
enum IpAddrKind {
    V4,
    V6,
}

fn main() {
    let ip_v4 = IpAddrKind::V4;
    let ip_v6 = IpAddrKind::V6;
}
```

Just like in C, you can associate a number with each enum variant. This number can later be retrieved by casting the enum value to a `usize`:

```rust
enum HttpStatus {
    Ok = 200,
    NotModified = 304,
    NotFound = 404,
}

fn main() {
    println!("{}", HttpStatus::Ok as usize); // 200
}
```

## Enums as a Union of Types

Unlike C, enums in Rust can include not only values but also various types (structs and tuples). In this case, an enum object will belong to one of these internal types. That is, by declaring an enum, we are not just declaring a list of possible values, but a list of types, one of which the enum object must belong to.

For example, an IPv4 address is encoded with 4 bytes, while an IPv6 address uses 16 bytes. Typically, an IPv4 address is written as four numbers ranging from 0 to 255, while an IPv6 address is usually written as a string. We can create an enum where the value is represented either as a tuple of 4 bytes or as a string:

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

fn main() {
    let home = IpAddr::V4(127, 0, 0, 1);
    let loopback = IpAddr::V6(String::from("::1"));
}
```

Now, let's look at an example of an enum using structs. We'll write a `Shape` enum consisting of two types: a square and a rectangle.

```rust
enum Shape {
    Square { width: f32 },
    Rectangle { width: f32, height: f32 }
}
fn main() {
    let square = Shape::Square { width: 4.0 };
}
```

One of the advantages of using enums is that they are very convenient to use with the `match` operator, as the compiler will force us to check all possible variants of the enumeration.

```rust
enum Shape {
    Square { width: f32 },
    Rectangle { width: f32, height: f32 }
}

fn calc_area(shape: &Shape) -> f32 {
    match shape { // Must check both Square and Rectangle
        Shape::Square { width } => width * width,
        Shape::Rectangle { width, height } => width * height,
    }
}

fn main() {
    let square = Shape::Square { width: 4.0 };
    println!("{}", calc_area(&square));
}
```

You can add methods to enums in the same way you do for structs:

```rust
enum Shape {
    Square { width: f32 },
    Rectangle { width: f32, height: f32 }
}

impl Shape {
    fn calc_area(&self) -> f32 {
        match self {
            Shape::Square { width } => width * width,
            Shape::Rectangle { width, height } => width * height,
        }
    }
}

fn main() {
    let square = Shape::Square { width: 4.0 };
    println!("{}", square.calc_area());
}
```

> [!NOTE]
> Enums in Rust are based on ADTs (Algebraic Data Types). This is a branch of theory that treats composite data types as combinations of sums and products of other types.

## if let

As mentioned, the `match` operator forces us to iterate through all possible enum variants. However, if we are only interested in one specific variant, we can use the **if let** construct—a version of the `if` operator using a destructuring pattern.

```rust,noplayground
if let шаблон = объект {
    // work with variables from the destructuring pattern here
}
```

For example, let's use `if let` to check if a `Shape` object is a square:

```rust
enum Shape {
    Square { width: f32 },
    Rectangle { width: f32, height: f32 }
}
fn main() {
    let s = Shape::Square { width: 4.0 };
    if let Shape::Square { width } = s {
        println!("This is square of width {width}");
    }
}
```

Of course, `if let`, like a regular `if`, can have an `else` branch.

## Memory Layout

Using enums, we can effectively store values of different types within an array.

```rust
enum MyEnum {
    Byte(u8),
    UInt(u32),
}

fn main() {
    let arr = [MyEnum::Byte(1), MyEnum::UInt(5)];
}
```

This is achieved because Rust reserves enough memory for an enum element to hold the largest possible variant of the enumeration. In other words, flexibility comes at the cost of potential memory overhead.

Additionally, besides the value itself, every enum object contains a **discriminant** — a number (ranging from `u8` to `u32` in size) that allows the program to identify which specific variant is currently stored in the object.

Массив из примера выше выглядит в памяти примерно так:

<pre class="ascii-diagram">
┏━━━━━━━━━━━━┯━━━━━━━━┯━━━━━━━━━━━┳━━━━━━━━━━━━┯━━━━━━━━━━━━━━━━━━━━━━━┓
┃discriminant┆u8 value┆empty space┃discriminant┆      u32 value        ┃
┗━━━━━━━━━━━━┷━━━━━━━━┷━━━━━━━━━━━┻━━━━━━━━━━━━┷━━━━━━━━━━━━━━━━━━━━━━━┛
</pre>

The `match` and `if let` operators check this discriminant to determine which internal type the enum object belongs to.

## C-style Enums Revisited

Now that we understand how enums are structured, we can take another look at the very first example in this chapter.

```rust
enum IpAddrKind {
    V4,
    V6,
}
```

From a semantic perspective, `V4` and `V6` are not just values, but unit-like variants. Consequently, in memory, objects of such enums are represented solely by their discriminant.

Furthermore, when we assign a specific numeric value to enum variants, we are simply defining the specific values for the discriminant.

```rust
enum HttpStatus {
    Ok = 200,          // discriminant = 200
    NotModified = 304, // discriminant = 304
    NotFound = 404,    // discriminant = 404
}
```

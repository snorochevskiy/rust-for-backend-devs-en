# Automatic Trait Implementation

As an introduction, let's consider the following example: we will try to create a struct to store the coordinates of a point in 2D space and attempt to compare two instances of the point for equality:

```rust,compile_fail
struct Point2D { x: i32, y: i32 }

fn main() {
  let p1 = Point2D {x: 1, y: 1};
  let p2 = Point2D {x: 1, y: 1};
  println!("p1 = p2: {}", p1 == p2);
}
```

When attempting to compile this code, we get an error:

```
error[E0369]: binary operation `==` cannot be applied to type `Point2D`
 --> src/my_module/num.rs:6:30
  |
6 |   println!("p1 = p2: {}", p1 == p2);
  |                           -- ^^ -- Point2D
  |                           |
  |                           Point2D
```

The compiler indicates that in order to compare objects for equality, their type must implement the [PartialEq](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html) trait, which provides a method for comparing two instances for equality.

> [!NOTE]
> Don't be confused by the name "Partial Equality": it is used specifically for equality comparison, and this "partiality" only comes into play in rare cases. For example, when comparing two `f32` values that are both `NaN`: the IEEE-754 specification requires that the result of comparing two `NaN`s be false, even though they are identical.

The `PartialEq` trait itself looks like this (it's actually a bit more complex, but we've captured the essence):

```rust
pub trait PartialEq {
    fn eq(&self, other: &Self) -> bool;
    fn ne(&self, other: &Self) -> bool { !self.eq(other) }
}
```

As we can see, the trait contains two methods: `eq` (equal) and `ne` (not equal). When the compiler encounters a comparison of two objects using the `==` operator, it replaces the use of the == operator with a call to the `eq` method. That is, the call `p1 == p2` will be replaced by `p1.eq(p2)`. Similarly, the use of the `!=` operator is replaced by a call to the `ne` method.

Thus, for the equality check to work for our `Point2D` struct, we must implement `PartialEq` for it:

```rust
struct Point2D { x: i32, y: i32 }

impl PartialEq for Point2D {
    fn eq(&self, other: &Self) -> bool {
        self.x == other.x && self.y == other.y
    }
}

fn main() {
  let p1 = Point2D {x: 1, y: 1};
  let p2 = Point2D {x: 1, y: 1};
  println!("p1 = p2: {}", p1 == p2);
}
```

Now everything works.

If we think about the implementation of the `eq` method for our point — which simply consists of comparing all fields — we'll notice that for the vast majority of structs we might write, comparison will also simply involve comparing all corresponding fields.

Fortunately, Rust has a special mechanism that allows generating these common trait implementations automatically — the `derive` attribute.

## derive

If we "attach" the `#[derive(PartialEq)]` attribute to our struct, the compiler will generate the PartialEq implementation for our type itself. The automatic implementation of the `eq` method simply compares all corresponding fields of the struct objects, which is exactly what we need.

```rust
#[derive(PartialEq)]
struct Point2D { x: i32, y: i32 }

fn main() {
  let p1 = Point2D {x: 1, y: 1};
  let p2 = Point2D {x: 1, y: 1};
  println!("p1 = p2: {}", p1 == p2);

  let p3 = Point2D {x: 0, y: 0};
  let p4 = Point2D {x: 1, y: 1};
  println!("p3 = p4: {}", p3 == p4);
}
```

As you can see, this implementation works correctly, while our code has become significantly shorter and more expressive.

> [!NOTE]
> You can see what implementation code is generated based on the derive attribute using the [cargo expand](https://crates.io/crates/cargo-expand) utility.

## Attributes

We've already discussed the `derive` attribute, but we aren't yet fully acquainted with attributes themselves.

An attribute is a special mark for the compiler that can be applied to a struct, function, module, etc.

An attribute has the following syntax:

```
#[attribute(attribute_arguments)]
```

When the compiler encounters an attribute, it turns to the corresponding handler, which performs some form of code generation or logic.

Accordingly, for the `derive` attribute, the Rust standard library has a special handler that generates a standard implementation for certain traits during compilation.

In addition to `PartialEq`, standard implementation generation is supported for a number of other traits from the standard library, for example:

* [Hash](https://doc.rust-lang.org/std/hash/trait.Hash.html) — a standard trait providing a method for calculating an object's hash code.
* [Debug](https://doc.rust-lang.org/std/fmt/trait.Debug.html) — a trait that declares a "debugging" method for string conversion. This is what's used when we print an object via `{:?}` in a `println!` call.
* [Default](https://doc.rust-lang.org/std/default/trait.Default.html) — allows creating a default value for many types. For example: 0 for numbers, false for booleans, an empty string for strings, and so on.

## The Clone Trait

We already know that due to the concept of ownership, when assigning an object from one variable to another, the object is moved.

However, if we want to copy the object instead of moving it, the [Clone](https://doc.rust-lang.org/std/clone/trait.Clone.html) trait comes to our rescue, providing the `clone()` method. The trait itself looks like this:

```rust,noplayground
trait Clone: Sized {
    fn clone(&self) -> Self;
}
```

You might wonder, what's so special about it? We could write a method ourselves that returns a copy of the object. For example, like this:

```rust
#[derive(Debug)]
struct Point2D { x: i32, y: i32 }

impl Point2D {
    fn make_clone(&self) -> Point2D {
        Point2D { x: self.x, y: self.y }
    }
}

fn main() {
    let p1 = Point2D { x: 1, y: 1};
    let p2 = p1.make_clone();

    println!("p1={:?}, p2={:?}", p1, p2);
   // Will print: p1=Point2D { x: 1, y: 1 }, p2=Point2D { x: 1, y: 1 }
}
```

However, the beauty is that for `Clone`, an implementation can also be generated simply by adding the `derive` attribute.

```rust
#[derive(Debug,Clone)]
struct Point2D { x: i32, y: i32 }

fn main() {
    let p1 = Point2D { x: 1, y: 1};
    let p2 = p1.clone();

    println!("p1={:?}, p2={:?}", p1, p2);
   // Напечатает: p1=Point2D { x: 1, y: 1 }, p2=Point2D { x: 1, y: 1 }
}
```

> [!NOTE]
> Automatic generation isn't the only reason to use the `Clone` trait. As we know from the previous chapter, polymorphic functions can impose constraints on which traits must be implemented by the argument type. Cloning is a very important functionality, so both in the standard library and in third-party libraries, there are a vast number of functions that impose the `impl Clone` constraint on their arguments.

Of course, if necessary, nothing prevents us from implementing `Clone` for our type manually.

```rust
#[derive(Debug)]
struct Point2D { x: i32, y: i32 }

impl Clone for Point2D {
    fn clone(&self) -> Point2D {
        Point2D {x: self.x, y: self.y}
    }
}

fn main() {
    let p1 = Point2D { x: 1, y: 1};
    let p2 = p1.clone();
    println!("p1={:?}, p2={:?}", p1, p2);
}
```

The `clone()` implementation generated using `#[derive(Clone)]` performs a deep copy of the object, i.e., the current object and all nested ones. This is why, if we want to apply `#[derive(Clone)]` to our struct, the types of all fields in that struct must also implement the `Clone` trait.

## The Copy Trait

From the [Ownership](ownership.md) chapter, we know that the assignment operation moves objects of composite types, while values of primitive types are simply copied.

In reality, to determine whether an object should be copied or moved, the compiler checks if the type implements the [Copy](https://doc.rust-lang.org/std/marker/trait.Copy.html) trait.

```rust,noplayground
trait Copy: Clone { }
```

As we can see, this trait inherits from the `Clone` trait but does not add any new methods. Such traits are called **marker traits**, meaning they simply serve as a "mark" for the compiler.

If the compiler sees that the type of the object used in an assignment operation implements the `Copy` trait, then instead of moving the object, it bitwise copies the value.

Let's rewrite our `Point2D` example, adding the `Copy` trait to the `derive` attribute.

```rust
#[derive(Debug,Clone,Copy)]
struct Point2D { x: i32, y: i32 }

fn main() {
  let p1 = Point2D { x: 1, y: 1};
  let p2 = p1; // p1 is copied into p2 instead of being moved

  println!("p1={:?}, p2={:?}", p1, p2);
  // p1=Point2D { x: 1, y: 1 }, p2=Point2D { x: 1, y: 1 }
}
```

Now, when assigning the variable `p1` to the variable `p2`, ownership transfer does not occur. Instead, a copy of the object from variable `p1` is created and assigned to variable `p2`.

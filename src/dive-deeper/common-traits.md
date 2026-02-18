# Core Traits

МWe have already encountered the following traits:

* [Debug](https://doc.rust-lang.org/std/fmt/trait.Debug.html) — defines functionality for obtaining a debug text representation of an object. It is used when outputting via the `println!` macro using the `{:?}` format specifier.
* [Clone](https://doc.rust-lang.org/std/clone/trait.Clone.html) — defines the `clone()` method, which creates a deep copy of an object.
* [Copy](https://doc.rust-lang.org/std/marker/trait.Copy.html) — a marker trait that tells the compiler that during assignment or passing to a function by value, a bitwise copy should be performed instead of moving ownership.
* [Hash](https://doc.rust-lang.org/std/hash/trait.Hash.html) — defines a method for computing a hash code from an object.
* [PartialEq](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html), [Eq](https://doc.rust-lang.org/std/cmp/trait.Eq.html) — define methods for comparing objects for equality.
* [Default](https://doc.rust-lang.org/std/default/trait.Default.html) — provides a constructor method that creates a default instance of an object.
* [Deref](https://doc.rust-lang.org/std/ops/trait.Deref.html) — allows taking a reference to data owned by an object. The compiler replaces the expression `&object` with `object.deref()`.
* [Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html) — allows iterating over the elements of an object.

In addition to those listed above, there are other widely used traits:

* [PartialOrd](https://app.gitbook.com/u/dkQPoX1YHFcD5pDtm1LhqZnDeAW2), [Ord](https://doc.rust-lang.org/std/cmp/trait.Ord.html) — declare methods for determining the ordering of two objects (used for sorting).
* [From](https://doc.rust-lang.org/std/convert/trait.From.html) / [Into](https://doc.rust-lang.org/std/convert/trait.Into.html) — define conversion from one type to another.
* [AsRef\<T>](https://doc.rust-lang.org/std/convert/trait.AsRef.html) — allows obtaining a reference to internal data given a reference to a parent object.
* [Borrow](https://doc.rust-lang.org/std/borrow/trait.Borrow.html) — allows borrowing a reference to a value owned by an object.
* [ToOwned](https://doc.rust-lang.org/std/borrow/trait.ToOwned.html) — allows obtaining an owned object from a reference (typically by cloning).
* [Drop](https://doc.rust-lang.org/std/ops/trait.Drop.html) — declares a destructor method that is called when an object goes out of scope.
* [Sized](https://doc.rust-lang.org/std/marker/trait.Sized.html) — a marker trait automatically implemented by the compiler if the size of the type is known at compile time.
* [Sync](https://doc.rust-lang.org/std/marker/trait.Sync.html) — a marker trait automatically added by the compiler to a type if it is safe to access a value of this type from multiple threads simultaneously. We will discuss this in more detail in the [Multithreading](multithreading.md) chapter.
* [Send](https://doc.rust-lang.org/std/marker/trait.Send.html) — a marker trait automatically added by the compiler for types whose values can be safely transferred to another thread. We will also discuss this in the [Multithreading](multithreading.md) chapter.

## A Closer Look at Eq and PartialEq

As mentioned earlier, there are two traits in Rust used for equality checks:

*   `PartialEq` — partial equality
    ```rust
    pub trait PartialEq<Rhs = Self> where Rhs: ?Sized {
        fn eq(&self, other: &Rhs) -> bool;        // operator ==
        fn ne(&self, other: &Rhs) -> bool { ... } // operator !=
    }
    ```
*   `Eq` — equality
    ```rust
    pub trait Eq: PartialEq { }
    ```

The `Eq` trait requires that the `eq` and `ne` methods (which correspond to the `==` and `!=` operators) satisfy the following properties:

* Reflexive: `a == a`
* Symmetric: if `a == b`, then `b == a`
* Transitive: if `a == b` and `b == c`, then `a == c`

The `PartialEq` trait does not have these strict requirements, so it can be used for types where "full" equality is not defined.

For example, `f32` and `f64` follow the IEEE 754 specification, which states that these types can contain a `NAN` (not a number) value. According to IEEE 754, `NAN != NAN`.

```rust
let a = f32::NAN;
let b = f32::NAN;
println!("{}", a == b) // false
```

Standard types that implement `Eq`:

* Integers: `i8` - `i128` and `u8` - `u128`
* Boolean type: `bool`
* Strings: `&str` and `String`
* Arrays, if the element type implements `Eq`
* `Vec<T>`, if `T` implements `Eq`
* The Unit type ()

Types that implement only `PartialEq`:

* Floating-point numbers: `f32`, `f64`.

## PartialOrd и Ord

These two traits are used to implement comparisons like "greater than, less than, or equal to".

Typically, implementing these traits is necessary to use objects in functionality that involves sorting values in ascending or descending order.

The [PartialOrd](https://doc.rust-lang.org/std/cmp/trait.PartialOrd.html) trait is used for comparing values that may not always be comparable.

```rust
pub trait PartialOrd<Rhs = Self>: PartialEq<Rhs> where Rhs: ?Sized {
    fn partial_cmp(&self, other: &Rhs) -> Option<Ordering>;
    // Default implementations
    fn lt(&self, other: &Rhs) -> bool { ... }
    fn le(&self, other: &Rhs) -> bool { ... }
    fn gt(&self, other: &Rhs) -> bool { ... }
    fn ge(&self, other: &Rhs) -> bool { ... }
}
```

As we can see, the primary method `partial_cmp` returns an `Option<Ordering>`.

```rust
pub enum Ordering {
    Less = -1, Equal = 0, Greater = 1,
}
```

If the values are comparable, `partial_cmp` returns `Some(Ordering)`.\
If the values are incomparable, `partial_cmp` returns `None`. We can look at the `f32` type again for an example:

```rust
println!("{:?}", 4.0.partial_cmp(&5.0));           // Some(Less)
println!("{:?}", f32::NAN.partial_cmp(&f32::NAN)); // None
```

Accordingly, the [Ord](https://doc.rust-lang.org/std/cmp/trait.Ord.html) trait is used to define a total (strict) ordering.

```rust
pub trait Ord: Eq + PartialOrd {
    fn cmp(&self, other: &Self) -> Ordering;
    // Default implementations
    fn max(self, other: Self) -> Self where Self: Sized { ... }
    fn min(self, other: Self) -> Self where Self: Sized { ... }
    fn clamp(self, min: Self, max: Self) -> Self where Self: Sized { ... }
}
```

For example, let's create a sea cargo type with two fields: weight and a fragile flag. We want to be able to sort a vector of cargo so that lighter items come before heavier ones, but all fragile items appear before all non-fragile items.

```rust
#[derive(Debug, PartialEq)]
struct Cargo {
    weight: f32,
    fragile: bool,
}

impl Eq for Cargo {}

impl PartialOrd for Cargo {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}

impl Ord for Cargo {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        if self.fragile && !other.fragile {
            return std::cmp::Ordering::Less;
        }
        if !self.fragile && other.fragile {
            return std::cmp::Ordering::Greater;
        }
        if self.weight < other.weight { std::cmp::Ordering::Less }
        else if self.weight > other.weight { std::cmp::Ordering::Greater }
        else { std::cmp::Ordering::Equal }
    }
}

fn main() {
    let mut v = vec![
        Cargo { weight: 2.0, fragile: false },
        Cargo { weight: 1.0, fragile: false },
        Cargo { weight: 3.0, fragile: true },
    ];
    v.sort(); // Requires the element type to implement Ord
    println!("{v:?}");
//Cargo{weight:3.0,fragile:true},Cargo{weight:1.0,fragile:false},Cargo{weight:2.0,fragile:false}
}
```

Usually, comparison logic is written in the `PartialOrd` implementation only if there are no plans to implement `Ord` for that same type. Otherwise, the `PartialOrd` implementation should simply call `Ord::cmp`, as shown in the example above.

## Default

This trait defines a constructor method that allows creating a default value for a type.

```rust
pub trait Default: Sized {
    fn default() -> Self;
}
```

The `Default` trait is implemented for many standard types:

```rust
fn main() {
    println!("{}", i32::default());          // 0
    println!("{}", f32::default());          // 0
    println!("{}", bool::default());         // false
    println!("{}", String::default());       // ""
    println!("{:?}", Vec::<i32>::default()); // []
}
```

Many functions in the standard library work with the `Default` trait. For example, the `Option` type has an u`nwrap_or_default()` method, which returns the default value if called on a `None` variant.

```rust
let o: Option<i32> = None;
let v = o.unwrap_or_default();
println!("{v}"); // 0
```

You can implement `Default` for your own type either by using the `derive` attribute:

```rust
#[derive(Debug, Default)]
struct Ip4Addr(u8, u8, u8, u8);

fn main() {
    let default_ip_addr = Ip4Addr::default();
    println!("{default_ip_addr:?}"); // Ip4Addr(0, 0, 0, 0)
}
```

Or manually:

```rust
#[derive(Debug)]
struct Ip4Addr(u8, u8, u8, u8);

impl Default for Ip4Addr {
    fn default() -> Self {
        Ip4Addr(127, 0, 0, 1)
    }
}

fn main() {
    let default_ip_addr = Ip4Addr::default();
    println!("{default_ip_addr:?}"); // Ip4Addr(127, 0, 0, 1)
}
```

## From and Into { #from-into }

The `From` and `Into` traits are two "sibling" traits used for converting between types.

The [From](https://doc.rust-lang.org/std/convert/trait.From.html) trait allows you to define how to create a type from another type `T`. The trait declares a `from` constructor method used to build one value from another:

```rust
pub trait From<T>: Sized {
    fn from(value: T) -> Self;
}
```

For example, let's implement `From` to create an "IPv4 address" struct from an array:

```rust
#[derive(Debug)]
struct Ip4Addr(u8, u8, u8, u8);

fn ping(addr: Ip4Addr) {
    println!("Ping {addr:?}");
}

impl From<[u8; 4]> for Ip4Addr {
    fn from(value: [u8; 4]) -> Self {
        let [a, b, c, d] = value;
        Ip4Addr(a, b, c, d)
    }
}

fn main() {
    let arr = [127, 0, 0, 1];
    let addr = Ip4Addr::from(arr);
    ping(addr);
}
```

The [Into](https://doc.rust-lang.org/std/convert/trait.Into.html) trait does the same thing, but from "the other side":

```rust
pub trait Into<T>: Sized {
    fn into(self) -> T;
}
```

While `From` is typically used as a constructor—to explicitly create an object of a desired type from another type — `Into` is often used when passing arguments in the form: arg: `impl Into<T>`. This allows the argument to be converted into the required type inside the function. For example:

```rust
#[derive(Debug)]
struct Ip4Addr(u8, u8, u8, u8);

fn ping(into_addr: impl Into<Ip4Addr>) { // Accept argument as Into
    println!("Ping {:?}", into_addr.into());
}

impl Into<Ip4Addr> for [u8; 4] {
    fn into(self) -> Ip4Addr {
        let [a, b, c, d] = self;
        Ip4Addr(a, b, c, d)
    }
}

fn main() {
    let arr = [127, 0, 0, 1];
    ping(arr);
}
```

However, the `Into` trait is rarely implemented explicitly. This is because when you implement `From<X> for Y`, you automatically get a "blanket implementation" of `Into<Y> for X`.

## AsRef

The [AsRef\<T>](https://doc.rust-lang.org/std/convert/trait.AsRef.html) trait declares the `as_ref()` method, which is used to obtain a reference from another reference — specifically, a reference to an internal field of an object or other data that the object owns.

```rust
pub trait AsRef<T> where T: ?Sized {
    fn as_ref(&self) -> &T;
}
```

This trait is used to retrieve a reference to data in the form it is already stored within the object. No data transformations are expected. Typically, this trait is used to specify function argument types as `impl AsRef<T>`.

Consider an example: we have a Shipment type that encapsulates a Product and an Address. We use `AsRef` to provide access to these internal fields.

```rust
struct Product(String);

struct Address(String);

struct Shipment {
    product: Product,
    address: Address,
}

impl AsRef<Product> for Shipment {
    fn as_ref(&self) -> &Product {
        &self.product
    }
}

impl AsRef<Address> for Shipment {
    fn as_ref(&self) -> &Address {
        &self.address
    }
}

fn process_product(prod: impl AsRef<Product>) {
    // ...
}

fn process_address(addr: impl AsRef<Address>) {
    // ...
}

fn main() {
    let s = Shipment {
        product: Product("laptop".to_string()),
        address: Address("In the middle of the nowhere".to_string()),
    };
    process_product(&s);
    process_address(&s);
}
```

> [!NOTE]
> Theoretically, we can use the `as_ref()` method outside the context of passing arguments to a function, simply by calling:
> 
> ```rust
> let addr: &Address = s.as_ref();
> ```
> 
> or even
> 
> ```rust
> let addr = AsRef::<Address>::as_ref(&s);
> ```
> 
> but in practice, this is rarely necessary.

In addition to `AsRef`, which provides an immutable reference, there is a corresponding trait [AsMut](https://doc.rust-lang.org/std/convert/trait.AsMut.html), which serves the same purpose but returns a mutable reference.

## Borrow

In terms of signature, the [Borrow](https://doc.rust-lang.org/std/borrow/trait.Borrow.html) trait is very similar to `AsRef`: it also serves to provide a reference to some internal object.

```rust
pub trait Borrow<Borrowed> where Borrowed: ?Sized {
    fn borrow(&self) -> &Borrowed;
}
```

The key difference is that all functionality defined in the standard library for the `Borrow` trait assumes a consistent relationship between the implementations of several traits for both the owning object and the reference obtained from it.

Specifically, for objects `x` and `y` of type `T` that implements `Borrow<R>`, it is expected that:

* If `T` and `R` implement the `Eq` trait, the expression `x.borrow() == y.borrow()` must return the same result as `x == y`.
* If `T` and `R` implement the `Ord` trait, the expression `x.borrow().cmp(&y.borrow())` must return the same result as `x.cmp(&y)`.
* If `T` and `R` implement the Hash trait, the hash code of `x` must be equal to the hash code of `x.borrow()`.

There is also a [BorrowMut](https://doc.rust-lang.org/std/borrow/trait.BorrowMut.html) trait, which returns a mutable reference.

## ToOwned

The [ToOwned](https://doc.rust-lang.org/std/borrow/trait.ToOwned.html) trait declares the `to_owned()` method, which allows you to obtain a new owned object from a reference.

```rust
pub trait ToOwned {
    type Owned: Borrow<Self>;

    fn to_owned(&self) -> Self::Owned;

    // Реализация по умолчанию
    fn clone_into(&self, target: &mut Self::Owned) { ... }
}
```

Often, `ToOwned` does the same thing as `Clone`, but in some situations, it returns a different type.

For example, the implementation of ToOwned for `&str` returns a `String` object.

```rust
let slice: &str = "aaa";
let owned: String = slice.to_owned();
```

## Drop

The [Drop](https://doc.rust-lang.org/std/ops/trait.Drop.html) trait is used to define functionality that should be executed when a value goes out of scope. As a rule, this trait is used to implement a destructor.

For types that capture an I/O resource, `Drop` can be used to release that resource. For types that own an object on the heap, `Drop` is used to free the memory.

For example, let's create a structure — a primitive analog of `Box` that encapsulates a pointer to the heap, and use `Drop` to free the allocated memory.

```rust
struct MyBox<T> {
    ptr: *mut T,
}

impl <T> MyBox<T> {
    fn new(val: T) -> MyBox<T> {
        let ptr = unsafe {
            let layout = std::alloc::Layout::for_value(&val);
            let ptr = std::alloc::alloc(layout) as *mut T;
            *ptr = val;
            ptr
        };
        MyBox {
            ptr
        }
    }
    fn get(&self) -> &T {
        unsafe { self.ptr.as_ref().unwrap() }
    }
    fn set(&self, new_val: T) {
        unsafe { *self.ptr = new_val; }
    }
}

impl<T> Drop for MyBox<T> {
    fn drop(&mut self) {
        unsafe {
            std::alloc::dealloc(self.ptr as *mut u8, std::alloc::Layout::new::<T>());
        }
        println!("Released memory");
    }
}

fn main() {
    {
        let my_box = MyBox::new(5);
        println!("Boxed num: {}", my_box.get());
    } // my_box goes out of scope
    println!("Box memory is already released here");
}
```

The program prints:

```
Boxed num: 5
Released memory
Box memory is already released here
```

> [!NOTE]
> Do not focus too much on the heap management in this example (especially since it is usually not handled quite like this). There is a high probability that when writing backends in Rust, you will never have to work with the heap directly.

## Sized

The [Sized](https://doc.rust-lang.org/std/marker/trait.Sized.html) trait is a marker trait that is automatically implemented by the compiler for types whose size is known at compile time.

An example of a non-`Sized` type is the str type. Not `&str`, but specifically `str`. Yes, the type of a string literal is actually `str`, but we always work with them via a slice reference `&str`. And a slice reference, as we know, has a known and constant size (two fields: a pointer and a length). Thus, `str` is not `Sized`, but `&str` is `Sized`.

Similarly to `str`, the `[T]` type — a sequence of elements in memory of an indeterminate size is also a non-`Sized` type. Just like with `str`, we work with `[T]` via the slice `&[T]`, which is `Sized`.

Another example of a non-`Sized` type is a trait object (`dyn Trait`). This is precisely why we work with trait objects either by reference `&dyn Trait` or by wrapping them in a smart pointer like `Box<dyn Trait>`.

At the same time, the `Vec` type is `Sized`: although the size of the buffer stored on the heap is not known at compile time, from the type system's perspective, the `Vec` type consists only of the part stored on the stack, and its size is known. Similarly, the `String` type is also `Sized`.

As we can see, all types that we can create ourselves using safe Rust (without unsafe and manual pointer manipulation) are `Sized`, even if they store data on the heap.

***

It should also be mentioned that by default, the compiler sets an implicit `Sized` bound for all generic type arguments.

That is, when we write code like this:

```rust
fn my_func<T>() { ... }
```

The compiler interprets it as:

```rust
fn my_func<T: Sized>() { ... }
```

Sometimes this restriction needs to be relaxed, for which an explicit `?Sized` trait bound is specified:

```rust
fn my_func<T: ?Sized>() { ... }
```

When browsing generic code in third-party libraries or the standard library, you will often encounter this relaxation of the trait bound via `?Sized`.


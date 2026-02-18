# Generics

Generics are a mechanism that allows you to write functionality that works with values while abstracting away the specific types of those values.

For example, if we are creating a "list" data structure, we focus on the operations performed on the elements in the list (adding an element, removing, searching, inserting), rather than the specific type of those elements. The functionality of a list doesn't change whether we are storing numbers or strings.

The `Vec` type, which we are already familiar with, is a generic type: we can store values of any data type within a vector.

## Generic Types and Functions

To understand generics, let's create the simplest possible generic type—a wrapper structure. This wrapper doesn't have any special functionality; it simply holds a value, but it is capable of holding a value of any type.

```rust
struct Holder<T> {
    v: T
}
```

Here, `<T>` is what's known as a **generic type argument**: the type we are abstracting over.

> [!NOTE]
> In the example above, we named our abstracted type `T` (the most common name for a generic type argument), but this name can be anything, such as "Element".
> 
> ```rust
> struct Holder<Element> {
>     v: Element
> }
> ```

`Holder<T>` — is less of a concrete type and more of a template for creating a type. When the compiler encounters the use of `Holder<T>` with a specific type, such as `i32`, it generates the concrete type `Holder<i32>`.

```rust
struct Holder<T> {
    v: T
}

fn main() {
    let bool_holder: Holder<bool> = Holder { v: true };
    let i32_holder: Holder<i32> = Holder { v: 5 };
    let string_holder: Holder<String> = Holder { v: "aaa".to_string() };
}
```

---

Not only structures but also functions can be generic. Let's write a corresponding generic constructor function for our `Holder<T>` structure:

```rust
fn make_holder<T>(v: T) -> Holder<T> {
    Holder {v: v}
}
```

As you can see, the generic argument for a function is specified the same way as for a structure: in angle brackets after the name.

Now we can create instances of `Holder` using this function:

```rust
struct Holder<T> {
    v: T
}

fn make_holder<T>(v: T) -> Holder<T> {
    Holder {v: v}
}

fn main() {
  let bool_holder: Holder<bool> = make_holder(true);
  let i32_holder: Holder<i32> = make_holder(5);
  let string_holder: Holder<String> = make_holder("aaa".to_string());
}
```

> [!NOTE]
> Note that the generic type argument `T` in `the make_holder<T>` function is not inherently linked to the generic argument `T` in the `Holder<T>` structure declaration. As mentioned before, `T` is simply the most popular name. If we wrote fn `make_holder<A>(v: A) -> Holder<A>`, the result would be exactly the same.

---

To complete the picture, let's look at how methods for generic structures are defined. We'll create two methods: get — to retrieve the value from our wrapper, and set — to write a new value.

```rust,noplayground
impl<T> Holder<T> {
    fn get(&self) -> &T {
        &self.v
    }
    fn set(&mut self, new_v: T) {
        self.v = new_v;
    }
}
```

In this construction, we see the generic type argument twice: in `impl<T>` and in `Holder<T>`. The one in `impl<T>` <ins>declares</ins> the name of the generic type argument for the entire `impl` block. The generic argument in `Holder<T>` establishes the link between the argument declared in `impl<T>` and the generic argument in the `Holder<T>` structure definition.

It is as if we are saying:

> For some type `T`, we are implementing the template type `Holder`, which, when parameterized by this type `T`, has these implementations for the `get` and `set` methods.

It might seem a bit confusing at first, but it will become clearer in more complex examples later.

And now, putting it all together:

```rust
struct Holder<T> {
    v: T
}

impl<T> Holder<T> {
    fn get(&self) -> &T {
        &self.v
    }
    fn set(&mut self, new_v: T) {
        self.v = new_v;
    }
}

fn make_holder<T>(v: T) -> Holder<T> {
    Holder {v: v}
}

fn main() {
    let mut h = make_holder(1);
    println!("{}", h.get()); // 1
    h.set(5);
    println!("{}", h.get()); // 5
}
```

## Monomorphization of Generics

As previously mentioned, generic types like `Holder<T>` are templates; concrete types are produced when the generic is parameterized with a specific type, such as `Holder<i32>`.

Every time the compiler encounters the use of a generic type with a new type argument, it generates a concrete version of the generic for that specific type. This generation of concrete types from generic types is called **monomorphization**.

For example, if we used our `Holder<T>` generic for the types `i32` and `bool`, the compiler monomorphizes `Holder<T>` for `i32` and `bool` roughly like this (in reality, the internal names will be more complex):

```rust,noplayground
struct Holder_i32 {
    v: i32
}
struct Holder_bool {
    v: bool
}
```

The same process occurs with methods and functions: concrete versions of the functions are generated for each type argument used.

<table class="two-side">
<tr>
<td width="50%">

```rust,noplayground
fn make_holder<T>(v: T) -> Holder<T> {
    Holder { v: v }
}
```

</td>
<td width="50%">

```rust,noplayground
let h1 = make_holder(5);
let h2 = make_holder(true);
```

</td>
</tr>

<tr>
<td width="50%"><center>↓</center></td>
<td width="50%"><center>↓</center></td>
</tr>

<tr>
<td width="50%">

```rust,noplayground
fn make_holder_i32(v: i32) -> Holder_i32 {
    Holder_i32 { v: v }
}

fn make_holder_bool(v: bool) -> Holder_bool {
    Holder_bool { v: v }
}
```

</td>
<td width="50%">

```rust,noplayground
let h1 = make_holder_i32(5);
let h2 = make_holder_bool(true);
```

</td>
</tr>
</table>

> [!NOTE]
> Languages like Java and C# also have generics. However, a distinguishing feature of generics in those languages is "type erasure" during compilation. In other words, in Java, generics are primarily used for additional type checking at compile time.\
> After compilation, information about the generic type arguments disappears and is unavailable during program execution.\
> In Rust, generics are more similar to templates in C++, where monomorphization occurs during compilation.

## Turbofish ::<>

We have already seen an example of using a generic function to create a generic struct object.

```rust
struct Holder<T> {
    v: T
}

fn make_holder<T>(v: T) -> Holder<T> {
    Holder {v: v}
}

fn main() {
    let mut h = make_holder(1); // variable type is Holder<i32>
}
```

In this case, the compiler was able to independently infer the type of the generic function `make_holder` based on the type of the argument the function was called with: since the argument type is `i32`, the function result type is also `Holder<i32>`.

However, if a function has no arguments, the compiler cannot infer the generic type on its own, and it must be specified explicitly. For example, consider a function that returns an empty vector:

```rust,noplayground
fn make_empty_vec<T>() -> Vec<T> {
    Vec::new()
}
```

When calling `make_empty_vec`, the generic type can be specified via the type of the variable to which the result is assigned:

```rust,noplayground
let v: Vec<i32> = make_empty_vec();
```

Alternatively, you can provide the generic type argument directly at the function call:

```rust,noplayground
let v = make_empty_vec::<i32>();
```

This syntax for providing a generic type argument, `::<>`, is nicknamed the turbofish.

## Generic Traits { #generic-traits }

Not only structures and functions can be generic; traits can be as well.

```rust
// Defines an interface for accessing a value inside a type
// using get and set methods.
// The type of the element being accessed is defined by a generic.
trait CanBeAccessed<T> {
    fn get(&self) -> &T;
    fn set(&mut self, new_v: T);
}

// Defines an interface for creating a new object.
trait HasGenericConstructor<T> {
    // Creates an object of the type implementing HasGenericConstructor<T>
    // from an object of type T.
    fn new(value: T) -> Self;
}

struct Holder<T> {
    v: T
}

impl<T> CanBeAccessed<T> for Holder<T> {
    fn get(&self) -> &T {
        &self.v
    }
    fn set(&mut self, new_v: T) {
        self.v = new_v;
    }
}

impl<T> HasGenericConstructor<T> for Holder<T> {
    fn new(value: T) -> Self {
        Holder { v: value }
    }
}

fn main() {
    let mut h = Holder::new(5);
    h.set(7);
    println!("{}", h.get());
}
```

As we can see from the example, generic traits behave exactly like regular ones, except they now have a generic type argument.

## Associated Types

For generic traits, there is an alternative syntax for specifying the generic type argument: through a special field called an **associated type**.

```rust,noplayground
trait Name {
    type AssociatedType;
}
```

The following example shows how a generic with a type argument compares to a generic with an associated type.

<table>
<tr>
<td width="50%" style="border: none;">
Trait with a generic type argument

```rust,noplayground
trait Трэйт<A> {
    ...
}


struct S {
    ...
}

impl Трэйт<i32> for S {
    ...

}
```

</td>
<td width="50%" style="border: none;">
Trait with an associated type

```rust,noplayground
trait Трэйт {
    type A;
    ...
}

struct S {
    ...
}

impl Трэйт for S {
    type A = i32;
    ...
}
```

</td>
</tr>
</table>

Let's rewrite our example from the generic traits section using an associated type:

```rust
trait CanBeAccessed {
    type ElementType;
    fn get(&self) -> &Self::ElementType;
    fn set(&mut self, new_v: Self::ElementType);
}

trait HasGenericConstructor {
    type TypeArg;
    fn new(value: Self::TypeArg) -> Self;
}

struct Holder<T> {
    v: T
}

impl<T> CanBeAccessed for Holder<T> {
    type ElementType = T;
    fn get(&self) -> &T {
        &self.v
    }
    fn set(&mut self, new_v: T) {
        self.v = new_v;
    }
}

impl<T> HasGenericConstructor for Holder<T> {
    type TypeArg = T;
    fn new(value: T) -> Self {
        Holder { v: value }
    }
}

fn main() {
    let mut h = Holder::new(5);
    h.set(7);
    println!("{}", h.get());
}
```

---

Although generic traits and traits with associated types serve the same purpose, they have two key differences.

1\) The first difference lies in <ins>where the type is defined</ins> to parameterize the generic:

* For a generic trait, the actual type of the generic argument is determined in the code where the structure implementing the trait is <ins>used</ins>.

* For a trait with an associated type, the type of the associated field is determined in the <ins>implementation block</ins> (`impl Trait for Type`). In the example above, in the block `impl<T> CanBeAccessed for Holder<T>`, we defined that the associated type for the CanBeAccessed trait will be the same as the generic type argument in the `Holder<T>` struct. This cannot be overridden at the site where the `Holder` object is created.

2\) The second difference lies in the <ins>implementation of traits for types</ins>.

The same generic trait can be implemented multiple times for the same type by specifying different types for the type argument:

```rust
trait ValueProducer<T> {
    fn produce() -> T;
}

struct ProducerImpl;

impl ValueProducer<i32> for ProducerImpl {
    fn produce() -> i32 {
        5
    }
}

impl ValueProducer<String> for ProducerImpl {
    fn produce() -> String {
        "Hello".to_string()
    }
}

fn main() {
    let n: i32 = ProducerImpl::produce();
    println!("{n}");

    let s: String = ProducerImpl::produce();
    println!("{s}");
}
```

It is impossible to write analogous code using a trait with an associated type, as you can only implement the trait once for a specific type.


## Trait Bounds { #generics-border }

When we declare a generic type or function, we can specify that the generic type argument must implement a certain trait. In other words, we can bound it by a trait.

```rust,noplayground
fn my_func<T: Trait>(...) -> ... {
    ...
}

struct MyStruct<T: Trait> {
    v: T
}
```

In this case, the compiler will only allow our generic type/function to be parameterized with types that implement the specified trait.

```rust,compile_fail
// The function takes an argument of any type that implements the Copy trait
fn duplicate<T: Copy>(v: T) -> T {
    v
}

fn main() {
    let num = 5;
    let num_dup = duplicate(num); // i32 implements the Copy trait

    let s = "Hello".to_string();
    let s_dup = duplicate(s); // String does not implement the Copy trait
             // ^^^^^^^^^^^^ the trait `Copy` is not implemented for `String
}
```

If a generic type argument must implement several traits, they should be listed using the + sign:

```rust,noplayground
fn my_func<T: Trait1 + Trait2 + Trait3>(...) -> ... {
    ...
}
```

At this point, it might seem that bounding types by traits is very similar to passing arguments using `impl Trait`, which we encountered in the chapter [Traits / Static Dispatch](traits.md#static-dispatching). Essentially, `impl Trait` can also only be replaced by a type that implements that trait.

<table>
<tr>
<td width="50%" style="border: none;">

Generic with a bound

```rust,noplayground
fn my_func<T: Трэйт>(v: T) -> ... {
    ...
}
```

</td>
<td width="50%" style="border: none;">

`impl Trait`

```rust,noplayground
fn my_func(v: impl Трэйт) -> ... {
    ...
}
```

</td>
</tr>
</table>


In other words: passing arguments via a generic with a bound and passing arguments via `impl Trait` are often just different syntaxes for the same underlying mechanism.

To demonstrate this, let's rewrite the example used in the [Traits / Static Dispatch](traits.md#static-dispatching)chapter using generic bounds.

> <details>
>   <summary>Original example with traits</summary>
> 
> ```rust
> trait CanIntroduce {
>     fn introduce(&self) -> String;
> }
> 
> struct Person { name: String }
> struct Dog { name: String }
> 
> impl CanIntroduce for Person {
>     fn introduce(&self) -> String {
>         format!("Hello, I'm {}", self.name)
>     }
> }
> 
> impl CanIntroduce for Dog {
>     fn introduce(&self) -> String {
>         String::from("Waf-waf")
>     }
> }
> 
> fn print_introduction(v: &impl CanIntroduce) {
>     println!("{}", v.introduce());
> }
> 
> fn main() {
>     let person = Person { name: String::from("John") };
>     let dog    = Dog    { name: String::from("Bark") };
> 
>     print_introduction(&person); // Hello, I'm John
>     print_introduction(&dog);    // Waf-waf
> }
> ```
> </details>


```rust
trait CanIntroduce {
    fn introduce(&self) -> String;
}

struct Person { name: String }
struct Dog    { name: String }

fn create_person(name: String) -> Person {
    Person { name }
}

fn create_dog(name: String) -> Dog {
    Dog { name }
}

impl CanIntroduce for Person {
    fn introduce(&self) -> String {
        format!("Hello, I'm {}", self.name)
    }
}

impl CanIntroduce for Dog {
    fn introduce(&self) -> String {
        String::from("Waf-waf")
    }
}

fn print_introduction<T: CanIntroduce>(v: T) { // Отличие здесь
    println!("{}", v.introduce ());
}

fn main() {
    let person = create_person("John".to_string());
    let dog    = create_dog("Bark".to_string());

    print_introduction(person); // Hello, I'm John
    print_introduction(dog);    // Waf-waf
}
```

As you can see, this minimal difference in syntax leads to the same result.

---

However, there are situations where only a generic with a bound can be used. For example, the compiler does not allow you to write a closure that returns `impl Trait` in some contexts, but it will allow a closure that returns a generic with a bound:

```rust
use std::fmt::Display;

fn produce_number() -> i32 {
    5
}

// This is not allowed:
// fn print_produced(f: fn() -> impl Format) {
//     println!("{}", f());
// }

fn print_produced<R: Display>(f: fn() -> R) {
    println!("{}", f());
}

fn main() {
    print_produced(produce_number);
}
```

Additionally, the `impl Trait` syntax has significant limitations when describing generic asynchronous functions, which we will explore later.


## where

There is an alternative syntax for specifying generic bounds — the **where** clause.

<table>
<tr>
<td width="50%" style="border: none;">
Without a where clause

```rust,noplayground
fn my_func<A: Трэйт1, B: Трэйт2>(
    аргумент1: A, аргумент2: B,
) -> Тип {
    ...

}
```

</td>
<td width="50%" style="border: none;">
With a where clause

```rust,noplayground
fn my_func<A, B>(
    аргумент1: A, аргумент2: B,
) -> Тип
where A: Трэйт1, B: Трэйт2 {
    ...
}
```

</td>
</tr>
</table>

The `where` clause makes the function signature more readable by preventing generic type-argument declarations from becoming too cluttered.

Let’s rewrite the `print_introduction` function from the previous example using a `where` clause:

```rust,noplayground
fn print_introduction<T>(v: T) where T: CanIntroduce {
    println!("{}", v.introduce ());
}
```

The `where` clause can be used not only with functions but also with types:

<table>
<tr>
<td width="50%" style="border: none;">

Generic with a bound

```rust,noplayground
struct Holder<T: Clone> {
    v: T
}
```

</td>
<td width="50%" style="border: none;">

Using a `where` clause

```rust,noplayground
struct Holder<T> where T: Clone {
    v: T
}
```

</td>
</tr>
</table>

The `where` clause is particularly useful for describing generics that contain other generics within their bounds. Consider this example:

```rust
use std::fmt::Display;

fn print_produced<F,R>(mut f: F)
    where
        F: FnMut() -> R,
        R: Display
{
    println!("{}", f());
}

fn main() {
    let sequence_producer = {
        let mut counter = 0;
        move || {
            counter += 1;
            counter
        }
    };
    
    print_produced(sequence_producer);
}
```

Here, the `print_produced` function takes an `FnMut` closure as an argument, which returns a value of any type that implements the `Display` trait. Attempting to write this function using the `impl Trait` syntax would result in a compilation error:

```rust,noplayground
fn print_produced(mut f: impl FnMut() -> impl Display) {
    println!("{}", f());
}
```

Error:

```
`impl Trait` is not allowed in the return type of `Fn` trait bounds
`impl Trait` is only allowed in arguments and return types of functions and methods
```


## Trait Specialization

For generic structs, you can define methods that are only available when the struct is monomorphized with a specific type. This allows you to add extra functionality for certain types.

Let’s add an `inc` (increment) method to our `Holder`, which will only be available if the `Holder` stores an `i32`.

```rust
struct Holder<T> {
    v: T
}

impl Holder<i32> {
    fn inc(&mut self) {
        self.v += 1;
    }
}

fn main() {
    let mut h = Holder { v: 1 };
    h.inc();
}
```

You can specialize a generic implementation not only for a specific type but also for a trait. Let's create a method that is available only if the `Holder` is parameterized with a type that implements the `Clone` trait.

```rust
struct Holder<T> {
    v: T
}

impl<T: Clone> Holder<T> {
    fn clone_value(&self) -> T {
        self.v.clone()
    }
}

fn main() {
    let h: Holder<String> = Holder { v: "text".to_string() };
    let s2: String = h.clone_value();
}
```

## const Generics

Generics allow you to parameterize types and functions not only with types but also with **constants**.

For example, an array type in Rust consists of two parts: the element type and the array size.

```rust
let arr: [i32; 3] = [1, 2, 3];
```

Therefore, if we want to write a generic function that returns an array of a specific size, we must specify a constant for the array size in addition to the generic type-argument for the elements. In generics, a constant is also defined within angle brackets, but unlike a generic type-argument, it is prefixed with the `const` keyword and includes the constant's type.

`<const CONSTANT: ConstantType>`

As an example, let’s write a function that creates an array of a given size and initializes its elements with a provided value:

```rust
/// Creates an array of the specified size, where all elements
/// are initialized with the given value.
fn make_array<T: Copy, const SIZE: usize>(init_value: T) -> [T; SIZE] {
    [init_value; SIZE]
}

fn main() {
    let arr: [i32; 5] = make_array::<i32, 5>(1);
    println!("{arr:?}"); // [1, 1, 1, 1, 1]
}
```

---

A generic constant value can be accessed just like a regular constant. For instance, let’s write a function that prints a passed value to the console multiple times. We will define the number of repetitions using a `const` generic argument.

```rust
use std::fmt::Display;

fn print_times<const QUANTITY: usize>(v: impl Display) {
    for _ in 0 .. QUANTITY {
        println!("{v}");
    }
}

fn main() {
    print_times::<3>("Hello");
}
```


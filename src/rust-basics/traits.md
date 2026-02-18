# Traits

Traits are used to define an interface for interacting with a type by describing the methods that must be present for it. If a type needs to implement a trait, it must define implementations for <ins>all</ins> methods declared in that trait.

> [!TIP]
> Java and C# programmers can draw parallels with interfaces. For C++ programmers, the closest analogy would be an abstract class.

Syntax for declaring a trait:

```rust,noplayground
trait TraitName {
    fn method_1(&self) -> ReturnType1;
    ...
    fn method_N(&self) -> ReturnTypeN;
}
```

Syntax for implementing a trait for a struct:

```rust,noplayground
impl TraitName for SomeStruct {
    fn method_1(&self) -> ReturnType1 { ... }
    ...
    fn method_N(&self) -> ReturnTypeN { ... }
}
```

Пример:

```rust
// A trait that indicates the type implementing it can introduce itself
trait CanIntroduce {
    fn introduce(&self) -> String;
}

struct Person {
    name: String
}

impl CanIntroduce for Person {
    fn introduce(&self) -> String {
        // A person makes an introduction by stating the name
        format!("Hello, I'm {}", self.name)
    }
}

fn main() {
    let person = Person { name: String::from("John") };

    println!("{}", person.introduce ()); // Hello, I'm John
}
```

## Polymorphism

Of course, traits aren't just for implementation's sake. Their primary use is to enable <ins>polymorphic code</ins> — code that interacts with types not directly, but through the traits they implement. For example, you can create a function that accepts an object of any type, as long as it implements a specific trait.

In Rust, there are two fundamentally different approaches for passing arguments based on traits:

* Static dispatch — when the argument type is specified as `impl Трэйт`
* Dynamic dispatch — when the argument type is specified as `dyn Трэйт`

If you program in C++, you likely already understand what this means. Otherwise, let's look at each of these two types.

### Static Dispatch { #static-dispatching }

First, we will look at the syntax for passing arguments using static dispatch, and then we'll break down how this mechanism works internally.

For a function to accept an argument by trait using static dispatch, you must specify the argument type as `impl Трэйт`.

Let's look at an example. We'll write a function that takes any type implementing the `CanIntroduce` trait from the example above and prints the "introduction" to the console:

```rust,noplayground
fn print_introduction(v: &impl CanIntroduce) {
    // All we know about v: its type implements the CanIntroduce trait
    println!("Value says: {}", v.introduce());
}
```

Since we are accepting the argument by its trait, the only things we can do with it are what is declared in the trait. But that is exactly what we need.

To demonstrate how this function can accept different types, let's create a `Dog` struct in addition to `Person` and implement `CanIntroduce` for it as well.

```rust,noplayground
struct Dog {
    name: String
}

impl CanIntroduce for Dog {
    fn introduce(&self) -> String {
        // Regardless of its name, a dog can only bark
        String::from("Waf-waf")
    }
}
```

Now let's see how the `print_introduction` function can be called for both `Person` and `Dog`.

```rust
# trait CanIntroduce {
#     fn introduce(&self) -> String;
# }
# 
# struct Person {
#     name: String
# }
# 
# impl CanIntroduce for Person {
#     fn introduce(&self) -> String {
#         format!("Hello, I'm {}", self.name)
#     }
# }
# 
# struct Dog {
#     name: String
# }
# 
# impl CanIntroduce for Dog {
#     fn introduce(&self) -> String {
#         // Вне зависимости от своего имени, собака может только погавкать
#         String::from("Waf-waf")
#     }
# }
# 
# fn print_introduction(v: &impl CanIntroduce) {
#     // Всё что мы знаем о v: его тип реализует трэйт CanIntroduce
#     println!("Value says: {}", v.introduce());
# }
# 
fn main() {
    let person = Person { name: String::from("John") };
    let dog    = Dog    { name: String::from("Bark") };

    print_introduction(&person); // Value says: Hello, I'm John
    print_introduction(&dog);    // Value says: Waf-waf
}
```

Everything works: we have written a polymorphic function that accepts an argument of any type that implements the trait.

---

Now let's talk about how this works and why the dispatch is called "static."

When the compiler encounters a function call using an `impl Trait` argument, it generates a specific version of that function for the concrete type used in the call.

> [!NOTE]
> This process of generating a function with a concrete type instead of a trait is called **monomorphization**.

Upon finding the call to `print_introduction` for `Person`, and then for `Dog`, the compiler will generate something like this (names will differ, obviously):

```rust,noplayground
fn print_introduction_$Person(v: &Person) {
    println!("Value says: {}", v.introduce());
}
fn print_introduction_$Dog(v: &Dog) {
    println!("Value says: {}", v.introduce());
}
```

Then, the compiler replaces the calls to the polymorphic `print_introduction` with calls to these specific versions:

```rust,noplayground
fn main() {
    let person = Person { name: String::from("John") };
    let dog    = Dog    { name: String::from("Bark") };

    print_introduction_$Person(&person);
    print_introduction_$Dog(&dog);
}
```

Thus, at compile time, each generated version of `print_introduction` knows exactly which type it is working with. Consequently, it knows the address of the correct `introduce` method for that specific type.

![](img/traits_static_dispatch.svg)

Since method addresses are static (known at the time the program is built), this is called static dispatch.

### Dynamic Dispatch { #dynamic_dispatching }

In contrast to static dispatch, there is dynamic dispatch. At first glance, the code difference is minimal: you simply replace `impl Trait` with `dyn Trait`. However, the underlying implementation is very different. As before, we'll look at the syntax first.

Here is what `print_introduction` looks like with dynamic dispatch:

```rust,noplayground
fn print_introduction(v: &dyn CanIntroduce) {
    println!("Value says: {}", v.introduce());
}
```

The calls to this function for `Person` and `Dog` remain unchanged:

```rust
# trait CanIntroduce {
#     fn introduce(&self) -> String;
# }
# 
# struct Person {
#     name: String
# }
# 
# impl CanIntroduce for Person {
#     fn introduce(&self) -> String {
#         format!("Hello, I'm {}", self.name)
#     }
# }
# 
# struct Dog {
#     name: String
# }
# 
# impl CanIntroduce for Dog {
#     fn introduce(&self) -> String {
#         String::from("Waf-waf")
#     }
# }
# 
# fn print_introduction(v: &dyn CanIntroduce) {
#     println!("Value says: {}", v.introduce());
# }
# 
fn main() {
    let person = Person { name: String::from("John") };
    let dog    = Dog    { name: String::from("Bark") };

    print_introduction(&person); // Value says: Hello, I'm John
    print_introduction(&dog);    // Value says: Waf-waf
}
```

While the external difference is minimal, internally, the compiler generates <ins>only one version</ins> of the function for dynamic dispatch, regardless of how many types use it.

How does `print_introduction` know which implementation of `CanIntroduce` to use? The argument `&dyn CanIntroduce` is not just a reference to the object; it is a <ins>pair of references</ins>. The first points to the object itself, and the second points to a **vtable** (virtual method table) for that specific type. In English literature, this pair of pointers is called a **fat pointer**.

When the compiler sees code where a concrete type is accessed via `dyn Trait`, it generates a vtable for that type. This table stores the names of the methods the type implements for the trait and the addresses of those implementations in the code segment. Effectively, the vtable is a directory mapping trait method names to concrete implementations.

At the call site of a function with a `dyn Trait` argument, the compiler generates code that passes this pair: the object address and its vtable address.

Inside the function itself, the compiler inserts code that:

1) Looks up the method's implementation address in the vtable by its name.
2) Calls that method.

![](img/traits_dynamic_dispatch.svg)

An object of type `dyn Trait` is called a **trait object**.

> [!TIP]
> Do not confuse:
> * `dyn Трэйт` (trait object) — a value of unknown type and unknown size.
> * `&dyn Трэйт` (reference to a trait object) — a pair of pointers: to the value and to the vtable.
> 
> Just as a slice reference `&[T]` is not just an address but an address + length, an `&dyn` reference is not just an address but an address + vtable address.

### impl vs dyn

Let's summarize. There are two ways to refer to a type through a trait it implements:

* `impl Трэйт` — replaced by a concrete type during compilation.
* `dyn Трэйт` — replaced by a trait object that proxies method calls to the real type using dynamic dispatch.

> [!NOTE]
> If you are not familiar with C++, the details of dynamic dispatch might be hard to grasp initially. Don't feel pressured to understand everything at once; return to this topic after you've mastered the basics of Rust.

At this stage, we don't have enough knowledge to cover all aspects of trait objects, but we can say the following:

* Static dispatch is **faster** because calls are made directly to method addresses, whereas dynamic dispatch requires a vtable lookup first.
* With `impl Trait`, arguments can be passed by reference (`&impl Trait`) or by value (`impl Trait`) because the size is known at compile time.
* `dyn Trait` <ins>cannot be passed by value</ins>. The compiler must know the exact size of all function arguments in machine code. Since the concrete type of a trait object is unknown, its size is also unknown. Therefore, trait objects are always passed via reference (`&dyn Trait`) or a smart pointer like `Box<dyn Trait>`.

## Implementing Traits for "Foreign" Types { #impl-for-foreign-types }

Unlike OOP languages where methods are defined inside the class body, Rust defines methods outside the struct body. This allows you to implement your own trait for "foreign" structs (those located in other libraries).

```rust
trait CanIntroduce {
    fn introduce(&self) -> String;
}

impl CanIntroduce for &str {
    fn introduce(&self) -> String {
        String::from("I am string slice")
    }
}

impl CanIntroduce for i32 {
    fn introduce(&self) -> String {
        String::from("I am integer")
    }
}

fn print_introduction(v: impl CanIntroduce) {
    println!("Value says: {}", v.introduce());
}

fn main() {
    print_introduction("a"); // Value says: I am string slice
    print_introduction(5);   // Value says: I am integer
}
```

However, Rust enforces the **"Orphan Rule"**, which states:

> Трэйт можно реализовать для типа только в том случае, если либо трэйт, либо тип (либо оба) принадлежит библиотеке в которой осуществляется реализация.

This means that even though `i32` belongs to the standard library, we could implement `CanIntroduce` for it only because `CanIntroduce` was declared by us in our program. We cannot implement a trait from a foreign library for a type from a foreign library. Either the trait or the type must belong to our `main.rs` or its modules.

In the [Newtype паттенр](../dive-deeper/newtype-pattern.md) chapter, we will look at a way to bypass Orphan Rule restrictions.

## Returning Traits from Functions { #return-impl-trait }

Rust allows you to not only pass arguments via traits but also return traits from functions.

The principle is the same as with arguments:

* If the return type is `impl Trait`, the compiler replaces it with a concrete type.
* If the return type is `dyn Trait`, the compiler creates a trait object.

For example:

```rust,noplayground
fn make_person() -> impl CanIntroduce {
    Person { name: String::from("John") }
}
```

However, there is a catch: since the compiler replaces `impl Trait` with a concrete type, you <ins>cannot return two different types</ins> from such a function.

Consider this example:

```rust,compile_fail
# trait CanIntroduce {
#     fn introduce(&self) -> String;
# }
# 
# struct Person {
#     name: String
# }
# 
# impl CanIntroduce for Person {
#     fn introduce(&self) -> String {
#         format!("Hello, I'm {}", self.name)
#     }
# }
# 
# struct Dog {
#     name: String
# }
# 
# impl CanIntroduce for Dog {
#     fn introduce(&self) -> String {
#         String::from("Waf-waf")
#     }
# }
# 
fn make_someone(is_person: bool) -> impl CanIntroduce {
    if is_person {
        Person { name: String::from("John") }
    } else {
        Dog { name: String::from("Bark") }
    }
}
# 
# fn main() {
#     let p = make_someone(true);
# }
```

This fails because if `impl CanIntroduce` is replaced by `Person`, returning a `Dog` becomes impossible, and vice versa.

Returning `dyn Trait` works perfectly in this situation::

```rust,noplayground
fn make_someone(is_person: bool) -> Box<dyn CanIntroduce> {
    if is_person {
        Box::new(Person { name: String::from("John") })
    } else {
        Box::new(Dog { name: String::from("Bark") })
    }
}

fn main() {
    let person = make_someone(true);
    let dog    = make_someone(false);

    print_introduction(person.as_ref());
    print_introduction(dog.as_ref());
}
```

In this example, we see a new type: Box. This is essentially a safe wrapper around a pointer. Box::new(value) moves a value from the stack to the heap and returns a Box containing the heap address. We will discuss this in detail in the [Smart Pointers](smart-pointers.md) chapter..

> [!TIP]
> If you know C++, you can think of `Box<T>` as being equivalent to `std::unique_ptr<T>`

The main reason we use `Box<dyn Trait>` instead of `&dyn Trait` is that we cannot create an object on the stack within a function and then return a reference to it. Once the function exits, its stack frame is cleared, and any reference to stack objects becomes invalid. Thus, we move the object to the heap and return a pointer (the Box).

## Default Method Implementations

Methods in a trait can have default implementations.

```rust
trait CanIntroduce {
    fn say_name(&self) -> String;
    fn introduce(&self) -> String { // default implementation
        format!("Hello, I am {}", self.say_name())
    }
}

struct Person {
    name: String
}

impl CanIntroduce for Person {
    fn say_name(&self) -> String {
        self.name.clone()
    }
}

fn main() {
    let person = Person { name: String::from("John") };
    // Calling the default implementation of the introduce method
    println!("{}", person.introduce()); // Hello, I am John
}
```

Default methods can be overridden just like any other trait method:

```rust
# trait CanIntroduce {
#     fn say_name(&self) -> String;
#     fn introduce(&self) -> String { // реализация по умолчанию
#         format!("Hello, I am {}", self.say_name())
#     }
# }
# 
# struct Person {
#     name: String
# }
# 
impl CanIntroduce for Person {
    fn say_name(&self) -> String {
        self.name.clone()
    }
    fn introduce(&self) -> String { // overriding the default
        format!("Hi, I'am {}", self.say_name())
    }
}
# 
# fn main() {
#     let person = Person { name: String::from("John") };
#     println!("{}", person.introduce());
# }
```

## Trait "Inheritance"

In Rust, one trait can "inherit" from another trait.

```rust,noplayground
trait A : B { ... } // Trait A "inherits" from Trait B
```

In practice, this means that if we want to implement trait `A` for a type, we <ins>must</ins> also implement trait `B` for that same type.

Example:

```rust
trait HasName {
    fn say_name(&self) -> String;
}

// Anyone implementing CanIntroduce must also implement HasName
trait CanIntroduce : HasName {
    fn introduce(&self) -> String;
}

struct Person {
    name: String
}

impl CanIntroduce for Person {
    fn introduce(&self) -> String {
        format!("Hello, I am {}", self.say_name())
    }
}

// The compiler requires a HasName implementation
// because Person implements CanIntroduce
impl HasName for Person {
    fn say_name(&self) -> String {
        self.name.clone()
    }
}

fn main() {
    let person = Person { name: String::from("John") };
    println!("{}", person.introduce()); // Hello, I am John
}
```

## Multi-Trait Bounds

Passing arguments via `impl Trait` can be seen as applying a constraint on the types that can be passed to the function.

```rust
fn print_introduction(v: impl CanIntroduce) { ... }
```

This constrains the function to only accept types implementing `CanIntroduce`. If you need an argument to implement <ins>multiple</ins> traits, use the `+` sign.

```rust
trait CanIntroduce { ... }
trait HasJob { ... }

fn print_worker_introduction(v: &(impl CanIntroduce + HasJob)) {
```

Note: this syntax is rarely used for complex bounds. Instead, another syntax is used, which we will cover in the [Generics](generics.md) chapter.

### Self

Often, a trait declaration needs to refer to the concrete type that will eventually implement the trait. For this, we use the keyword `Self` (capitalized).

For example, let's create a trait for a default constructor:

```rust,noplayground
trait HasDefaultConstructor {
    fn make_default() -> Self;
}
```

When writing this trait, we don't know what type `make_default` will return yet—it depends on the implementer. `Self` acts as a placeholder for that type.

Implementation for `Person`:

```rust
trait HasDefaultConstructor {
    fn make_default() -> Self;
}

struct Person {
    name: String
}

impl HasDefaultConstructor for Person {
    fn make_default() -> Self {
        Person { name: "Anonymous".to_string() }
    }
}

fn main() {
    let p: Person = Person::make_default();
    println!("Default name: {}", p.name);
}
```

Inside the `impl`, you can use either `Self` or the actual type name (`Person`). The effect is the same.

```rust,noplayground
impl HasDefaultConstructor for Person {
    fn make_default() -> Person { // Person instead of Self
        Person { name: "Anonymous".to_string() }
    }
}
```

## Requirements for Trait Objects

> [!NOTE]
> This section might be confusing now. It's best to re-read it after the [Generics](generics.md) chapter.

Not all traits can be turned into trait objects. For the compiler to generate `dyn Trait`, the trait must be <ins>object safe</ins>. This means:

*   The trait must not contain methods that return `Self` or take `Self` as an argument (except for the receiver `&self`).

    ```rust
    trait A {
        fn f(&self, other: &Self) -> Self;
    }
    ```

* The trait must not contain static methods (methods without `self`).
    ```rust
    trait A {
        fn f() -> i32;
    }
    # 
    # struct B {}
    # 
    # impl A for B {
    #     fn f() -> i32 {
    #         5
    #     }
    # }
    # 
    # fn call_f(a: &dyn A) {
    #     println!("Do nothing");
    # }
    # 
    # fn main() {
    #     let b = B {};
    #     call_f(&b);
    # }
    ```

*   The trait must not contain methods with their own generic type parameters. We'll talk about [Generics](generics.md) later. E.g., following trait cannot be converted to a trait object.

    ```rust
    trait A {
        fn f<T>();
    }
    ```

*   The trait must not contain associated types (which are kind of generics):

    ```rust
    trait A {
        type X;
    }
    ```

If a trait inherits from other traits, those must also be object safe.

## unsafe trait

If you create a trait containing methods that imply unsafe operations (like pointer manipulation), the trait itself should be declared with the `unsafe` keyword.

```rust
unsafe trait MyTrait {
    fn do_something_dangerous();
}
```

Implementing such a trait requires the `unsafe` keyword as well.

```rust
unsafe impl MyTrait for MyStruct {
    fn do_something_dangerous() {
        ...
    }
}
```

Implementing an `unsafe` trait doesn't mean you must use an `unsafe` block inside the methods. The primary purpose of marking a trait as `unsafe` is to warn implementers that the trait carries specific safety contracts they must uphold.


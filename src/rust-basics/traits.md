# Traits

Traits are used to define an interface for interacting with a type by describing the methods it must possess. If a trait is implemented for a certain type, you must define implementations for <ins>all</ins> methods declared in that trait.

> [!TIP]
> Java and C# programmers may draw parallels with interfaces. For C++ programmers, the closest analogy would be an abstract class.

Trait declaration syntax:

```rust,noplayground
trait Name {
    fn method_1(&self) -> Type1;
    ...
    fn method_N(&self) -> TypeN;
}
```

Syntax for implementing a trait for a struct:

```rust,noplayground
impl Trait for Struct {
    fn method_1(&self) -> Type1 { ... }
    ...
    fn method_N(&self) -> TypeN { ... }
}
```

Example:

```rust
// A trait indicating that a type implementing it can introduce itself
trait CanIntroduce {
    // The "introduce" method
    fn introduce(&self) -> String;
}

struct Person {
    name: String
}

impl CanIntroduce for Person {
    fn introduce(&self) -> String {
        // A person introduces themselves by stating their name
        format!("Hello, I'm {}", self.name)
    }
}

fn main() {
    let person = Person { name: String::from("John") };

    println!("{}", person.introduce ()); // Hello, I'm John
}
```

## Polymorphism

Of course, traits aren't used just for the sake of implementing them. The primary use of traits is the ability to write polymorphic code. This is code that interacts with types not directly, but through the traits they implement. For example, it allows you to create a function that takes an object of any type as an argument, as long as that type implements the required trait.

In Rust, there are two fundamentally different approaches to passing arguments based on traits:

* Static dispatch — when the argument type is specified as `impl Трэйт`
* Dynamic dispatch — when the argument type is specified as `dyn Трэйт`

If you program in C++, you likely already understand what this means. Otherwise, let's examine each of these two types in detail.

### Static Dispatching { #static-dispatching }

First, we will look at the syntax for passing arguments using static dispatch, and then we'll examine how this mechanism works under the hood.

To have a function accept an argument via a trait using static dispatch, you must specify the argument type as `impl Trait`.

Let's look at an example. We'll write a function that accepts any type implementing the `CanIntroduce` trait from the previous example and prints the "introduction" to the console:

```rust,noplayground
fn print_introduction(v: &impl CanIntroduce) {
    // Everything we know about v: its type implements the CanIntroduce trait
    println!("Value says: {}", v.introduce());
}
```

Of course, since we are accepting the argument by its trait, the only thing we can do with it is what is declared in the trait. But that is exactly what we need.

To demonstrate how this function can accept arguments of different types, let's create a `Dog` struct in addition to the `Person` struct and implement the `CanIntroduce` trait for it as well.

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

Now let's see how we can call the `print_introduction` function for both `Person` and `Dog`.

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
#         // Regardless of its name, a dog can only bark
#         String::from("Waf-waf")
#     }
# }
# 
# fn print_introduction(v: &impl CanIntroduce) {
#     // Everything we know about v: its type implements the CanIntroduce trait
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

Everything works: we have written a polymorphic function that accepts an argument of any type that implements the required trait.

---

Now let's talk about how this works and why the dispatch is called "static".

The reason is that when the compiler encounters the use of a function with an argument of type `impl Trait`, it generates a specific version of that function for the concrete type with which the function was called.

> [!NOTE]
> This process of generating a function with a concrete type instead of a trait is called **monomorphization**.

That is, after finding a call to the `print_introduction` function for `Person` and then for `Dog`, the compiler will generate something like the following (the actual internal names will differ, of course):

```rust,noplayground
fn print_introduction_$Person(v: &Person) {
    println!("Value says: {}", v.introduce());
}
fn print_introduction_$Dog(v: &Dog) {
    println!("Value says: {}", v.introduce());
}
```

Next, the compiler replaces the calls to the polymorphic `print_introduction` function with calls to these concrete versions:

```rust,noplayground
fn main() {
    let person = Person { name: String::from("John") };
    let dog    = Dog    { name: String::from("Bark") };

    print_introduction_$Person(&person);
    print_introduction_$Dog(&dog);
}
```

Thus, at the compilation stage, each generated version of the `print_introduction` function already knows exactly which concrete type it is working with. Consequently, the address of the required `introduce` method for that specific type is known to that version of `print_introduction`.

![](img/traits_static_dispatch.svg)

Method addresses are static (known at the time the program is built), which is why this dispatch is called static.

### Dynamic Dispatching { #dynamic_dispatching }

In contrast to static dispatch, there is also dynamic dispatch. At first glance, the syntactic difference between the two is minimal: you simply change the argument type from `impl Trait` to `dyn Trait`. However, the underlying implementation is fundamentally different. As we did before, let’s first examine the syntax and then dive into the internal mechanics.

Here is what the `print_introduction` function looks like using dynamic dispatch:

```rust,noplayground
fn print_introduction(v: &dyn CanIntroduce) {
    println!("Value says: {}", v.introduce());
}
```

Interestingly, the calls to this function for the `Person` and `Dog` types do not change at all:

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

As mentioned, the difference is only skin-deep. In static dispatch, the compiler generates a separate version of the function for every distinct type it is called with. In dynamic dispatch, only one version of the function exists.

How does `print_introduction` know which implementation of `CanIntroduce` to use or where to find the correct `introduce` method? The secret lies in the fact that the `&dyn CanIntroduce` argument isn't just a simple reference to the object. It is a pair of pointers: the first points to the object itself, and the second points to a **vtable** (virtual method table) for that specific type. In Rust terminology, this pair is known as a **fat pointer**.

When the compiler encounters code where a concrete type is accessed via `dyn Trait`, it generates a unique vtable for that type. This table contains the names of the methods the type implements for that trait and their respective memory addresses in the code segment. Effectively, the vtable acts as a directory mapping trait methods to their concrete implementations.

At every call site where a function expects a `dyn Trait` argument, the compiler generates code that passes this "fat pointer" (the object's address + the vtable address).

Inside the function itself, the compiler inserts logic to:

1) Look up the address of the method implementation in the vtable using the method name.
2) Jump to and execute that method.

![](img/traits_dynamic_dispatch.svg)

An object of type `dyn Trait` is officially called a **trait object**.

> [!TIP]
> Don't get them confused:
> * `dyn Trait` (trait object) — a value of unknown type and unknown size.
> * `&dyn Trait` (reference to a trait object) — a fat pointer consisting of two pointers: one to the value and one to the vtable.
> 
> Just as a slice reference `&[T]` is not just a pointer to the start of a sequence but a pointer plus a length, a `&dyn` reference is not just a pointer to data, but a pointer to data plus a pointer to a vtable.

### impl vs dyn

Let's summarize. There are two ways to refer to a type via a trait it implements:

* `impl Трэйт` — is replaced by a concrete type during compilation.
* `dyn Трэйт` — is replaced by a trait object that proxies method calls to the real type using dynamic dispatch.

> [!NOTE]
> If you are not familiar with C++, the details of dynamic dispatch will likely be difficult to understand. Don't try to grasp everything at once; just come back to this topic later once you are comfortable with basic Rust and ready for a deeper dive.

At this stage, we don't yet have enough knowledge to consider all aspects of using trait objects, but for now, we can say the following:

* Method calls via static dispatch are much faster because the call is made directly to the method's address, whereas with dynamic dispatch, the code must first find the method's address in a vtable.
* During compilation, `impl Trait` is simply substituted with a concrete type, so arguments of this type can be passed either by reference (`&impl Trait`) or by value (`impl Trait`).
* Unlike `impl Trait`, `dyn Trait` cannot be passed by value. This is because when a function is compiled into machine code, the exact size of all its arguments must be known. But with dynamic dispatch, the concrete type of the argument is unknown, and therefore its size is unknown as well. This is why trait objects are always passed either by reference — `&dyn Trait`, or through a smart pointer — `Box<dyn Trait>` (which we will discuss later).

## Implementing a Trait for "Foreign" Types { #impl-for-foreign-types }

Unlike OOP languages, where class methods are defined directly within the class body, in Rust, struct methods are defined outside the struct body. This allows you to implement your own trait for "foreign" structs (those located in other libraries).

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

However, Rust has a rule called the "Orphan rule," which states:

> You can implement a trait for a type only if either the trait or the type (or both) belongs to the library in which the implementation is defined.

In other words, even though the `i32` type belongs to the standard library, we were able to implement the `CanIntroduce` trait for it only because the `CanIntroduce` trait was declared by us in our own program. We cannot define a trait from a "foreign" library for a type from a "foreign" library. Either the trait or the type must belong to our `main.rs` or its modules.

In the [Newtype pattern](../dive-deeper/newtype-pattern.md) chapter, we will look at a way to bypass the limitations of the Orphan rule.

## Returning a Trait from a Function { #return-impl-trait }

Rust allows you to not only pass arguments via a trait but also to return a trait from a function.

The principle is the same as with passing arguments:

* If the return type is `impl Trait`, the compiler will replace it with a concrete type.
* If the return type is `dyn Trait`, the compiler will create a trait object.

For example:

```rust,noplayground
fn make_person() -> impl CanIntroduce {
    Person { name: String::from("John") }
}
```

However, there is a catch: since the compiler simply replaces `impl Trait` with a specific concrete type, you cannot return two different types from such a function.

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
#         // Regardless of its name, a dog can only bark
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

As we know, `impl Trait` is simply replaced by the compiler with a concrete type. Therefore, if the return type `impl CanIntroduce` is replaced by, for instance, `Person`, returning a `Dog` object becomes impossible. And vice versa.

Returning `dyn Trait` works perfectly in this situation:

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

In this example, we encounter a type we haven't seen before — `Box`. Essentially, it is a safe wrapper around a pointer. The constructor `Box::new(value)` moves the value from the stack to the heap and returns a `Box` object, which contains a pointer to the address of the object in the heap. We will cover `Box` in more detail in the [Smart Pointers](smart-pointers.md) chapter.

> [!TIP]
> If you are familiar with C++, you can think of `Box<T>` as being the same as `std::unique_ptr<T>`.

The primary reason we use `Box<dyn Trait>` instead of `&dyn Trait` is that we cannot create an object on the stack within a function and then return a reference to it. When the function finishes, its stack frame is cleared along with all the objects inside it, making any reference to those objects invalid. This is why we move the object to the heap and return a pointer (`Box`) to that object in the heap.

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
    // Call the default implementation of the introduce method
    println!("{}", person.introduce()); // Hello, I am John
}
```

Methods with a default implementation are overridden in the same way as regular trait methods:

```rust
# trait CanIntroduce {
#     fn say_name(&self) -> String;
#     fn introduce(&self) -> String { // default implementation
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
    fn introduce(&self) -> String { // overriding
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

In Rust, one trait can "inherit" from another trait. (In technical terms, these are often called Supertraits).

```rust,noplayground
trait A : B { ... } // Trait A "inherits" trait B
```

In practice, this means that if we want to implement trait `A` for our type, we are also required to implement trait `B` for it.

For example:

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

// The compiler will require an implementation for HasName
// since it found an implementation for CanIntroduce
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

## Specifying Multiple Trait Bounds

Let’s look at passing arguments via `impl Трэйт` from a different angle. When we specify a trait as a function argument, we are effectively imposing a bound (restriction) on the types that can be passed to that function.

For example, by declaring an argument like this:

```rust
fn print_introduction(v: impl CanIntroduce) { ... }
```

we impose a constraint: the function can only be called with an argument whose type implements the `CanIntroduce` trait. But what if we need the argument to implement two traits? In that case, you simply list the required traits separated by the `+` sign.

```rust
trait CanIntroduce { ... }
trait HasJob { ... }

fn print_worker_introduction(v: &(impl CanIntroduce + HasJob)) {
```

It is worth noting that this specific syntax for specifying multiple trait bounds is rarely used in this form. Instead, developers typically use a different syntax which we will explore in the [Generics](generics.md) chapter.

### The Self Keyword

It is quite common to need a way to refer to the concrete type for which a trait is being implemented within the trait definition itself. For this purpose, Rust uses the Self keyword (with a capital "S").

For example, let’s create a trait that declares a type must have a default constructor function:

```rust,noplayground
trait HasDefaultConstructor {
    fn make_default() -> Self;
}
```

When we write this trait, we don't yet know what type the `make_default` function will return, as it depends entirely on the type implementing the trait. Therefore, we cannot specify a concrete type name for the return value. This is exactly where `Self` comes to the rescue.

Now, let’s implement this trait for the `Person` struct from our previous examples:

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

In this implementation of `HasDefaultConstructor` for `Person`, we specified the return type of `make_default` as `Self`. However, we could have also used `Person` explicitly: the effect is identical.

```rust,noplayground
impl HasDefaultConstructor for Person {
    fn make_default() -> Person { // Person instead of Self
        Person { name: "Anonymous".to_string() }
    }
}
```

## Trait Object Requirements

> [!NOTE]
> This section might be difficult to understand right now. It is best to reread it after completing the chapter on [Generics](generics.md).

Not all traits can be turned into trait objects. For the compiler to generate a trait object (i.e., `dyn Trait`), the trait must be object-safe. A trait is object-safe if it satisfies the following requirements:

*   The trait must not contain methods that return `Self` or take `Self` as an argument.

    ```rust
    trait A {
        fn f(&self, other: &Self) -> Self;
    }
    ```

* The trait must not contain static methods (methods that do not take a `self` parameter):
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

*   The trait must not contain methods that declare new generic type parameters that are not associated with the generic parameters defined at the trait level. We will discuss [generics](generics.md) in another chapter. For example, a trait object cannot be created for a trait like this:

    ```rust
    trait A {
        fn f<T>();
    }
    ```

*   The trait must not contain associated types (which are a variety of generics):

    ```rust
    trait A {
        type X;
    }
    ```

If a trait inherits from other traits (supertraits), all inherited traits must also satisfy the requirements mentioned above to be convertible into a trait object.

## Unsafe Traits

If we create a trait containing methods that imply _unsafe_ operations, such as working with raw pointers, the trait should be declared using the `unsafe` keyword.

```rust
unsafe trait MyTrait {
    fn do_something_dangerous();
}
```

To implement such a trait, you must also use the `unsafe` keyword.

```rust
unsafe impl MyTrait for MyStruct {
    fn do_something_dangerous() {
        ...
    }
}
```

The fact that we are implementing an unsafe trait does not mean we are required to use an `unsafe` block inside the methods. An unsafe trait might not even contain unsafe methods (as we will see later with the `Send` and `Sync` traits). The sole purpose of marking a trait as `unsafe` is to highlight the potential danger of the trait to whoever implements it for their types.


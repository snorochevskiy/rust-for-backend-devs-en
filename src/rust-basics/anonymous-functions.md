# Anonymous Functions

Rust has been influenced by functional programming, which is why it supports anonymous functions, closures, and higher-order functions.

An anonymous function is a function that acts as an object; it can be assigned to a variable, passed as an argument to another function, and, of course, called.

Anonymous functions are declared using the following syntax:

```
|argument_1, ..., argument_n| -> ReturnType { function_body }
```

> [!TIP]
> The source code of an anonymous function is often referred to as a "function literal."

For example:

```rust
fn main() {
    // Create an anonymous function and assign it to a variable
    let inc: fn(i32) -> i32 = |x: i32| { x + 1 };
    let a = 1;
    // Call our function exactly like a regular one
    let b = inc(a);
    println!("{b}"); // 2
}
```

Note that the type of an anonymous function looks like this:

```
fn(argument_type_1, ..., argument_type_n) -> return_type
```

This type is called a **function pointer**.

As with regular variables, the type of an anonymous function can be omitted if the compiler is able to infer it from the function body.

```rust
let inc = |x: i32| { x + 1 };
```

Additionally, curly braces around the function body are optional if it consists of only a single expression:

```rust
let inc = |x: i32| x + 1;
```

In some situations, the compiler can also infer the argument types. In such cases, you can omit the types entirely:

```rust
fn main () {
    // The compiler infers the type of x based on the usage of inc below.
    let inc = |x| x + 1;

    let a = 1;
    let b = inc(a);
    println!("{b}")
}
```

> [!NOTE]
> Anonymous functions are often called "lambda expressions," referring to the branch of mathematics known as "Lambda Calculus," which serves as the foundation for functional programming. They are also frequently called "lambda functions."

## Higher-Order Functions

A higher-order function is a function that either takes other functions as arguments or returns a function as its result.

For example, let's write a higher-order function named `transform` that takes two arguments — a number and a function — and returns the result of applying that function to the number:

```rust
fn transform(a: i32, f: fn(i32) -> i32) -> i32 {
    f(a)
}

fn main() {
    let inc: fn(i32) -> i32 = |x: i32| { x + 1 };
    let a = 9;
    let b = transform(a, inc);
    println!("{b}"); // 10
}
```

Now let's look at an example where a function returns an anonymous function as its result:

```rust
fn create_inc() -> fn(i32) -> i32 {
    |x: i32| x + 1 
}

fn main() {
    let inc = create_inc();
    let a = 1;
    let b = inc(a);
    println!("{b}"); // 2
}
```

## Function Pointers

Anonymous functions are compiled in the same way as regular functions and are placed in the code segment. When we assign an anonymous function to a variable, we are simply storing a pointer to the function located in the code segment.

In that case, how does an anonymous function fundamentally differ from a regular one at the executable code level? It doesn't. Furthermore, a pointer to a regular function can also be assigned to a variable:

```rust
fn func_inc(x: i32) -> i32 {
    x + 1
}

fn main() {
    let inc: fn(i32) -> i32 = func_inc;
    let a = inc(7);
    println!("{a}"); // 8
}
```

As we can see, anonymous functions are essentially just an alternative syntax for creating functions.

## Closures { #closure }

In the previous example, we created a `create_inc` function that returns an anonymous "increment" function:

```rust
fn create_inc() -> fn(i32) -> i32 {
    |x: i32| x + 1 
}
```

Now let's try to write a function that returns an anonymous function which increases its argument not by one, but by a specified "step":

```rust
fn make_inc_with_step(step: i32) -> fn(i32) -> i32 {
    |x| { x + step }
}
```

Unfortunately, this code will not compile and will produce an error:

```
|x| { x + step }
^^^^^^^^^^^^^^^^ expected fn pointer, found closure
```

The issue is that from within the body of our anonymous function, we are accessing data that belongs to a scope outside of the function. Such an anonymous function is called a **closure**.

> [!NOTE]
> The name comes from the fact that the anonymous function "closes over" its environment. It is also often said that a closure "captures" data from the outer context.

Closures are inherently more complex than "pure" anonymous functions (which depend only on their arguments). A pure anonymous function simply turns into a regular function in the code segment, and we interact with it via a function pointer. A closure, however, is an object representing a complex combination of code and captured data.

> [!NOTE]
> The concept of a "pure" function is widely used in functional programming. A pure function accesses only its arguments and does not interact with any data external to it.

Let's first rewrite the code so that it works, and then we will examine how closures are structured.

```rust
fn make_inc_with_step(step: i32) -> impl Fn(i32) -> i32 {
    move |x| { x + step }
}

fn main() {
    let inc_with_5 = make_inc_with_step(5);
    let a = inc_with_5(2);
    println!("{a}"); // 7
}
```

As you can see, two things have changed:

* `fn(i32)->i32` became `impl Fn(i32)->i32`, which explicitly indicates that a closure is not just a function pointer, but an object of some type that implements the `Fn` trait.

* The [**move**](https://doc.rust-lang.org/std/keyword.move.html) keyword appeared before the function literal declaration. This explicitly tells the compiler that if we use a value from the outer context inside the closure, the ownership of that value moves to the closure.

## Closure Types

As we mentioned, all "pure" anonymous functions use a single data type: a function pointer, which looks like `fn(...)->...`. More precisely, the specific types differ, but they are all grouped into the function pointer family. For closures, however, there are three different traits:

* [Fn](https://doc.rust-lang.org/std/ops/trait.Fn.html) — A trait for closures that capture values from the environment by value or by immutable reference. These closures are safe to use in a multithreaded environment because they only read the captured value.

* [FnMut](https://doc.rust-lang.org/std/ops/trait.FnMut.html) — A trait for closures that capture values from the environment and modify them via a mutable reference. These closures cannot be used in a multithreaded environment without prior synchronization, as they can change the captured value.

* [FnOnce](https://doc.rust-lang.org/std/ops/trait.FnOnce.html) — A trait for closures that capture values from the environment by value (i.e., take ownership) and destroy that value during the call. Such a closure can only be called once, because after the first call, the value is destroyed and unavailable for a second call.

## Capture by Reference and Moving Ownership

Как мы сказали выше, замыкание может захватывать значения из внешнего контекста по значению или по ссылке. От того, каким образом захваченное значение будет <ins>использовано</ins> в теле замыкания, будет зависеть то, на основе какого трэйта компилятор сгенерирует тип замыкания: `Fn`, `FnMut` или `FnOnce`.

Рассмотрим самый простой пример захвата по немутабельной ссылке.

```rust
fn main() {
    let salutation = "Hello".to_string();

    // Closure type: impl Fn(&str)->String
    let greet = |name: &str| make_greeting(&salutation, name);

    println!("{}", greet("John")); // Hello John

    // The variable `salutation` is captured by the closure via an immutable reference,
    // so it can still be used after the capture.
    print_string(salutation); // OK, data is still usable
}

fn make_greeting(salutation: &str, name: &str) -> String {
    format!("{} {}", &salutation, name)
}

fn print_string(s: String) {
    println!("{s}")
}
```

The capture of the `salutation` variable occurred via an immutable reference because we use the immutable reference `&salutation` inside the closure body.

> [!NOTE]
> We had to create separate functions `make_greeting` and `print_string` so that their signatures clearly show which arguments are used by reference and which by value. Using the `format!` and `println!` macros directly would compromise the integrity of the example, as these macros use string objects by reference even if you explicitly pass them by value.


If we use a variable captured from the external context by value in the closure body, then the capture will also occur by value.

```rust,compile_fail
fn main() {
    let salutation = "Hello".to_string();

    // Closure type: impl FnOnce(&str)->String
    let greet = |name: &str| make_greeting(salutation, name);

    println!("{}", greet("John")); // Hello John

    // Now that `salutation` is captured by value (i.e., moved into the closure),
    // it can no longer be used here.
    print_string(salutation); // Error: use of moved value: `salutation`
}

fn make_greeting(salutation: String, name: &str) -> String {
    format!("{} {}", &salutation, name)
}

fn print_string(s: String) {
    println!("{s}")
}
```

---

Why didn't we use the `move` keyword before the anonymous functions in the previous examples, as we did in the [Closure](#closure) section? The reason is that `move` only needs to be explicitly specified if the closure outlives the scope in which it was created. In our simple example, the compiler is able to figure out the expected behavior unambiguously. But if we try to move the creation of the `greet` closure into a separate function, `move` will be required:

```rust
fn make_greet_closure() -> impl Fn(&str) -> String {
    let salutation = "Hello".to_string();
    move |name: &str| make_greeting(&salutation, name)
}
```

Here, our closure outlives the scope where it was created: the scope (the body of the `make_greet_closure` function) ends, but the closure continues to live in the code that called `make_greet_closure`.

## FnMut

As mentioned, if a closure captures a value via a mutable reference, the compiler makes that closure an implementation of the `FnMut` trait.

As an example, let's write another incrementing closure. However, this time the closure will hold a reference to a number that stores the increment step. Furthermore, after each call, this increment step will increase by one.

```rust
fn main() {
    let mut step = 1;

    // impl FnMut(i32)->i32
    let mut growing_inc = |x: i32| {
        let step_ref = &mut step;
        let res = x + *step_ref;
        *step_ref += 1;
        res
    };
    println!("{}", growing_inc(1)); // 2
    println!("{}", growing_inc(1)); // 3
    println!("{}", growing_inc(1)); // 4
}
```

As you can see, with each call to the closure, the number by which the closure increases the passed argument grows by one.

> [!IMPORTANT]
> It is important to remember that the closure type depends not on how the value is captured from the outer scope, but on how that captured value is used.
> 
> The closure in the example above has a type based on FnMut not because it captures the value by mutable reference, but because it modifies a value whose lifetime is longer than a single closure call via a mutable reference.

Consider the same example, but now our closure will capture the `step` variable by value instead of by mutable reference.

```rust
fn main() {
    let mut step = 1;

    // impl FnMut(i32)->i32
    let mut growing_inc = |x: i32| {
        let res = x + step;
        step += 1;
        res
    };
    println!("{}", growing_inc(1)); // 2
    println!("{}", growing_inc(1)); // 3
    println!("{}", growing_inc(1)); // 4
}
```

This closure still increases the increment step after each call, and the closure type still implements the `FnMut` trait. Why? As we said: it doesn't matter how the closure captured the value; what matters is how it modifies it.

The closure captured the `step` variable by value and placed it in its context, where this value exists outside of the closure calls themselves. When called, the closure modifies this value via <ins>a mutable reference</ins>.

## FnOnce

Now that we've looked at `Fn` and `FnMut` in detail, understanding `FnOnce` should be straightforward. The main thing to remember is that the closure type depends on what the closure does with the captured value:

* `Fn` closures only read the captured value.
* `FnMut` closures modify the captured value via a mutable reference.
* `FnOnce` closures <ins>destroy</ins> (consume) the captured value.

Consider the following example:

```rust
// A wrapper around println!() that takes a string by value,
// therefore taking ownership from the calling code.
fn print_and_destroy(s: String) {
    println!("{s}");
}

fn main() {
    let text = "text".to_string();

    // Closure type: FnOnce()
    let closure_print_and_destroy = || print_and_destroy(text);

    closure_print_and_destroy();
}
```

If you try to call `closure_print_and_destroy()` a second time, the compiler will throw an error:

> closure cannot be invoked more than once because it moves the variable `text` out of its environment.

This type of closure is often used when a closure captures a resource where the operation has a side effect: closing a file, clearing memory, or other operations that can only be performed once.

## Concrete Closure Types

In our code, we do not have access to the specific type of a closure because that specific type is only generated during compilation.

That is, we cannot write:

```rust,noplayground
fn make_inc_with_step(step: i32) -> impl Fn(i32) -> i32 {
    move |x| { x + step }
}

fn main() {
    let inc_with_5: Какой-то тип = make_inc_with_step(5);
}
```

The only things we can know about a closure's type are which trait it implements and its signature (argument and return types).

> [!NOTE]
> In Rust nightly, you can enable aliases for `impl Trait` using the `type_alias_impl_trait` flag. Then you could write:
> 
> ```rust,noplayground
> type MyFn = impl Fn(i32) -> i32;
> let inc_with_5: MyFn = make_inc_with_step(5);
> ```
> 
> As of the release of Rust 1.92, this feature was still not available in the stable branch.


It is also very important to know that in a Rust program, no two closures have the same type. Even if two closures have the same signature and implement the same trait, the specific types the compiler infers for them will be different. Consequently, we cannot write code like this:

```rust
fn make_inc(is_decrement: bool) -> impl Fn(i32) -> i32 {
    if is_decrement {
        return move |x| { x - 1 };
    } else {
        return move |x| { x + 1 };
    }
}
```

As you might have guessed, compiling this function results in an error because impl Trait must be replaced by a concrete type during compilation, and these two closures have different concrete types.

As a solution, we can use the trick we learned in the chapter on [Traits](traits.md) — dynamic dispatch. That is, return `dyn Fn` instead of `impl Fn`.

```rust
fn make_inc(is_decrement: bool) -> Box<dyn Fn(i32) -> i32> {
    if is_decrement {
        Box::new(move |x| { x - 1 })
    } else {
        Box::new(move |x| { x + 1 })
    }
}
fn main() {
    let dec: Box<dyn Fn(i32) -> i32> = make_inc(true);
    let a = 2;
    let b = dec(a);
    println!("{b}"); // 1
}
```

This code compiles and runs without any issues.

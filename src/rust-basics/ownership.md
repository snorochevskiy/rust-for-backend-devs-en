# Ownership

Ownership is Rust's most distinct feature, allowing the language to provide performance comparable to languages with manual memory management without actually requiring it. Rust guarantees the absence of memory leaks and does so without a garbage collector. This result is achieved through the concept of **ownership**.

In Rust, any object of a non-primitive type must have exactly one owner. The **owner** is the variable to which the object is assigned. Therefore, when the value of one variable is assigned to another, ownership of the object moves from the first variable to the second, rendering the first variable invalid.

Consider this example:

```rust
fn main() {
  let s1 = String::from("some string");
  let s2 = s1; // ownership of the string moves from s1 to s2
  // At this point, s1 is no longer valid.

  println!("{}", s2); // You can now only work with s2, not s1
}
```

Attempting to access an invalid variable will result in a compilation error:

```rust,compile_fail
fn main() {
  let s1 = String::from("some string");
  let s2 = s1;

  println!("{}", s1);
// 2 |   let s1 = String::from("some string");
//   |       -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait
// 3 |   let s2 = s1;
//   |            -- value moved here
// 4 |
// 5 |   println!("{}", s1);
//   |                  ^^ value borrowed here after move
}
```

The fact that every object has only one owner allows the compiler to understand exactly where the memory occupied by the object should be cleared. This happens at the point where the owner of the object ceases to exist. Consequently, the compiler inserts code to call the destructor for the data owned by variables at the end of their scope.

This approach allows for memory management without a garbage collector while guaranteeing no leaks. The Rust compiler mechanism that tracks object lifetimes and ensures memory is cleared correctly — and that there are no references to cleared memory—is called the **borrow checker**.

## Ownership and Scopes

The lifetime of a variable is tied to the scope in which it is declared: when the scope ends, all variables within it disappear, and the memory they owned is cleared. However, a variable declared inside a scope can "give" its data to another variable declared outside that scope.

```rust
fn main() {
  let s1;
  {
    let s2 = String::from("some string");
    s1 = s2; // the value is given to a variable from the outer scope
  }
  println!("{s1}"); // OK
}
```

This example demonstrates how ownership transfer works when moving between scopes. We will need this information later when we will be learning [Lifetimes](lifetimes.md)

## Transferring Ownership

Ownership transfer occurs during assignment, but there are other scenarios as well. Here is a full list of operations where ownership is transferred:

* Assignment
* Passing an object to a function as an argument
* Returning a value from a function
* Capturing an object with a closure

Let’s look at ownership transfer during a function call:

```rust
fn main() {
    let name = String::from("Stas");

    // The string from 'name' moves into the function, making 'name' invalid
    let greeting = greet(name);
    // <- Here, 'name' can no longer be used

    println!("{}", greeting); // Hello Stas!!!
}

fn greet(name: String) -> String {
    // The string object returned by the format! call moves to the calling code
    format!("Hello {}!!!", name)
}
```

Now that we understand that passing a variable as an argument effectively destroys that variable (because ownership is moved), let’s look at an example that seems completely normal in most programming languages but may surprise someone learning Rust.

Suppose we want to write a program that prints a string to the console (it doesn’t matter where it comes from) and also prints the length of that string. To calculate the length, we’ll create a separate function that simply calls `len()` on the provided string.

```rust,compile_fail
fn len_of_string(s: String) -> usize {
    s.len()
}

fn main() {
    let s = String::from("aaa");
    let len = len_of_string(s);
    println!("{}", s); // <- variable `s` is already invalid here
}
```

Here we immediately run into a problem: we try to print the variable `s`, but it is no longer valid because it transferred ownership of its data when we called the function on the previous line.

How can we solve this problem? Purely for demonstration purposes, let’s remember that with tuples we can return multiple values from a function. That means we can return the object passed as an argument back to the caller.

```rust
fn len_of_string(s: String) -> (String, usize) {
    let length = s.len();
    (s, length)
}

fn main() {
    let s = String::from("aaa");
    let (s, len) = len_of_string(s);
    println!("Len of {s} is {len}"); // "Len of aaa is 3"
}
```

It looks strange, but it clearly demonstrates how ownership is moved during a function call.

Fortunately, Rust provides a much better mechanism: **borrowing**.

> [!NOTE]
> The standard library has a [drop](https://doc.rust-lang.org/std/mem/fn.drop.html) function that destroys an object. It works simply by taking ownership and doing nothing with it, allowing the object to go out of scope immediately.
> 
> ```rust
> let s = String::from("aaa");
> drop(s);
> ```
> 
> If you look at the implementation of `drop` (we are not yet familiar with [Generics](generics.md), so we cannot fully understand its signature), you will see that it does nothing:
> 
> ```rust
> pub fn drop<T>(_x: T) {}
> ```
> 
> The object is destroyed simply because the `drop` function takes ownership of it and does not pass it anywhere else.


## Borrowing

Instead of passing the variable’s value as an argument, we can pass a reference to that value. This way, ownership of the object remains with the variable, and we merely allow the function to use it temporarily.

```rust
fn len_of_string(s: &String) -> usize {
    s.len()
}

fn main() {
    let s = String::from("aaa");
    let len = len_of_string(&s);
    println!("Len of {s} is {len}");
}
```

Passing an argument by reference in Rust is called **borrowing**. The term reflects the idea that the function “borrows” the object for use and then returns it to the owner once it finishes.

> [!TIP]
> In a real program, we would not use `&String` as the argument type for `len_of_string`. Instead, we would use the slice type `&str`, which would allow the function to be called with both `String` values and string literals.\
> We wrote the function this way purely to make borrowing easier to explain.

## Referential Safety { #referential-safety }

From the previous section, we already know that in Rust we can take a reference to an object and pass it to a function.

But consider the following scenario:

1\) We create a vector with capacity for 3 elements and fill it with values.

```rust,noplayground
let mut vector: Vec<i32> = Vec::with_capacity(3);
vector.push(1);
vector.push(2);
vector.push(3);
```

![](img/ownership_vector.svg)

2\) Next, we take an immutable reference to the second element of the vector. This reference points directly to the memory address where the second element is stored.

```rust,noplayground
let reference: &i32 = &vector[1];
```

![](img/ownership_vector_ref.svg)

3\) The reference to the second element is still alive, but now we also take a mutable reference to the entire vector. Using it, we add another element to the vector. Since the vector’s buffer is already full, a new, larger buffer is allocated on the heap, all elements are copied into it, then the new element is added. The old buffer is then freed.

```rust,noplayground
let vec_ref = &mut vector;
vec_ref.push(4);
```

![](img/ownership_vector_realloc.svg)

The question is: what does the original reference — the one that pointed to the second element — now point to? Obviously, such a reference would become invalid.

Fortunately, Rust is a safe language, so the compiler will not allow you to write this code. When producing an error, it follows the rule of referential safety:

> At any point in the code, for any given object, there can be either exactly one mutable reference or any number of immutable references.

This rule also makes references safe in a multithreaded environment. If we are only reading data, it is safe to do so from any number of concurrent threads. However, any read operation becomes potentially unsafe if there is another thread that modifies the same data concurrently.

To better understand how reference checking works, let’s look at a simpler example:

```rust,compile_fail
fn main() {
    let mut s = String::from("x");

    let r1 = &mut s; // <-- taking a mutable reference
    let r2 = & s;    // <-- attempting to take an immutable reference

    println!("{r1}, {r2}");
}
```

The compiler will produce an error:

```
4 |   let r1 = &mut s; // <-- taking a mutable reference
  |            ^^^^^^ mutable borrow occurs here
5 |   let r2 = & s;    // <-- попытка взять немутабельную ссылку
  |            ^^^ immutable borrow occurs here
6 |
7 |   println!("{r1}, {r2}");
  |              ^^ mutable borrow later used here
```

## Ownership and Primitive Types

All the ownership rules described above do not apply to primitive types.

Primitive types occupy little memory and do not own additional resources. Therefore, in situations where ownership would be transferred for composite types, primitive types are simply copied.

```rust
fn increment(a: i32) -> i32 {
    a + 1
}

fn main() {
    let x = 5;
    let y = increment(x);
    println!("x={}, y={}", x, y);
}
```

## The for Loop and Ownership

Another interesting aspect to consider is how ownership works when iterating with a `for` loop.

Consider the following example:

```rust,compile_fail
fn main() {
    let arr = [String::from("1"), String::from("2"), String::from("3")];

    for n in arr {
        println!("{n}");
    }

    println!("{arr:?}");
}
```

This example will not compile because in the `for` loop, on each iteration, the next array element is assigned to the variable `n` for use inside the loop body. Assignment causes ownership to be transferred.

As a result, we effectively destroy the array simply by printing its elements.

This problem is solved just as easily as the earlier function argument issue — by taking a reference.

```rust
fn main() {
    let arr = [String::from("1"), String::from("2"), String::from("3")];

    for n in &arr {
        println!("{n}");
    }

    println!("{arr:?}");
}
```

Now that we replaced `arr` with `&arr` in the loop header, each iteration assigns not the next element itself to `n`, but a reference to it. Therefore, the array is not destroyed.

> [!NOTE]
> After this example, it might seem that writing Rust programs turns into a constant battle with the compiler. At first, that may indeed be the case. However, once you become accustomed to the concept of ownership, you will naturally start writing code the correct way.


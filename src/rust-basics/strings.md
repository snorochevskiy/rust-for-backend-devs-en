# Strings

> [!NOTE]
> If this chapter seems difficult upon first reading, don't try to master it completely right away. Feel free to come back to it after reading the chapters on [Ownership](ownership.md) and [Structs](structs.md).

Strings in Rust are stored as UTF-8 encoded character buffers. There are two primary string types in Rust that differ in how they interact with this buffer: `&str` and `String`.

## &str (string slice)

If we write a string literal in our code (a string in double quotes), that string has the type `&str`.

```rust
let s: &str = "some text";
```

Essentially, [&str](https://doc.rust-lang.org/std/primitive.str.html) is a slice pointing to a buffer containing a sequence of UTF-8 encoded characters.\
In some ways, these strings are similar to `const char*` strings in the C language, with the key difference being that, as a slice, `&str` stores both the starting memory address of the buffer and its length.

When the compiler encounters a string literal in the code, it typically places that string in the data segment (or code segment, depending on the target platform), where it "lives" from the start of the program until it ends.

The `&str` type does not manage the memory where the string resides; it simply references the data. This memory can belong to the data segment, the heap (as the property of a `String` object), or even reside on the stack.

## String

While `&str` is a slice that references a buffer without managing it, `String` is the owner of the buffer containing the string.

Technically, [String](https://doc.rust-lang.org/std/string/struct.String.html) is a wrapper around a `Vec<u8>` vector that stores a sequence of UTF-8 characters. Because of this, `String` always has sole ownership of its string buffer, and this buffer is always located on the heap.

For any `String`, you can always create an `&str` slice that references the buffer owned by that `String`.

There are three main ways to create a `String` variable:

* Using the `String::from` constructor.
* Using `String::new` to create an empty string and then filling it.
* Converting from `&str` using the `.to_string()` method.

### The String::from Constructor

The most straightforward way to create a String object is the String::from(&str) constructor function (we'll discuss these more in the [Structs](structs.md) chapter). This function:

1. Creates a `String` object, which—as mentioned — is a wrapper around a `Vec<u8>`.
2. Copies the string from the `&str` argument into the newly created vector.
3. Returns the initialized `String` object.

```rust
fn main() {
    // A slice pointing to a static string in the data segment
    let slice: &str = "text";

    // This creates a buffer on the heap and copies "text" into it.
    // On the stack, we get a triplet of values, just like a Vec:
    // the buffer address, total capacity, and the number of bytes filled.
    let s = String::from(slice);
}
```

> [!NOTE]
> Naturally, it's much shorter and simpler to just write `String::from("text")`. In the example above, we created the `slice` variable separately only for the sake of clarity.

### The String::new Constructor

The `String::new` constructor simply creates a new `String` object with an empty buffer. This is useful, for example, when you need to pass a `String` to a function that will fill it with text.

For instance, a function that reads a line from the console takes a mutable reference to a `String` object as a parameter to store the input.

```rust
fn main() {
    println!("Please enter some text and hit Enter button");

    let mut buf = String::new(); // Create an empty string
    std::io::stdin().read_line(&mut buf); // Read console input into buf

    println!("You have entered: {buf}");
}
```

You can also add characters to a `String` using the `push(char)` method, or append string slices using the `push_str(&str)` method.

```rust
fn main() {
    let mut s = String::new();
    s.push('H');
    s.push('e');
    s.push('l');
    s.push('l');
    s.push('o');
    s.push_str(" world!");

    println!("{s}"); // Hello world!
}
```

### The to_string() Method

The final way to create a `String` is by calling the `.to_string()` method on an `&str` slice. Essentially, this method does exactly the same thing as `String::from(&str)`, just with different syntax.

```rust
fn main() {
    let s: String = "text".to_string();
}
```

## &str and String in Memory

To summarize how `&str` and `String` are laid out in memory, let's look at the following example where we:

1) Create an `&str` slice from a string literal.
2) Create a `String` from that slice.
3) Create a slice that references the buffer owned by the `String`.

```rust
fn main() {
  // The compiler sees a constant string literal and places 
  // it in the static data segment.
  let a_slice_1: &str = "text";

  // Create a String from the characters pointed to by the slice.
  // This results in a copy of the string's characters on the heap.
  let a_string: String = String::from(a_slice_1);

  // Create a second slice that points to the buffer in the heap
  // owned by the String.
  let a_slice_2: &str = a_string.as_str();
}
```

The memory layout would look something like this:

![](img/string_in_memory.svg)

## The format! macro

We are already familiar with the `println!` macro used for console output. This macro takes a format string where values can be "injected" using `{}`.

The standard library also offers the `format!` macro. It works exactly like `println!`, but instead of printing the text to the console, it returns the result as a `String`.

```rust
fn main() {
    let s: String = format!("{} in the power of the 2 is {}", 3, 9);
    println!("{s}"); // 3 in the power of the 2 is 9
}
```


# Result

Let's look at another ubiquitous type — the [Result](https://doc.rust-lang.org/std/result/) enum.

```rust
enum Result<T, E> {
   Ok(T),
   Err(E),
}
```

As we can see, `Result` is somewhat similar to `Option`, but while Option stores either a value or "nothing," `Result` stores either a value or an error.

`Result` is used as the return type for functions that can end in an error.

Let's look at a simple example:

```rust
fn square_root(num: f32) -> Result<f32, String> {
    if num < 0.0 {
        Err("Cannot calculate for negative number".to_string())
    } else {
        Ok(num.sqrt())
    }
}

fn main() {
    println!("sqrt(-4) = {:?}", square_root(-4.0));
    // sqrt(-4) = Err("Cannot calculate for negative number")
    
    println!("sqrt(4) = {:?}", square_root(4.0));
    // sqrt(4) = Ok(2.0)
}
```

The `square_root` function calculates the square root of the argument passed to it. If the argument is non-negative, it returns `Ok(value)`, otherwise — `Err` (since you cannot take the square root of negative numbers in real-number arithmetic).

Extracting a value from a `Result` is very similar to extracting a value from an `Option`:

* Using the `unwrap` / `unwrap_or` methods
* Using the `match` operator
* Using the `if-let` operator

```rust
# fn square_root(num: f32) -> Result<f32, String> {
#     if num < 0.0 {
#         Err("Cannot calculate for negative number".to_string())
#     } else {
#         Ok(num.sqrt())
#     }
# }
# 
fn main() {
    match square_root(-4.0) {
        Ok(v) => println!("sqrt(-4)={v}"),
        Err(e) => println!("Cannot calculate sqrt(-4): {e}"),
    }
    
    let v = square_root(4.0).unwrap_or(0.0);
    println!("sqrt(4)={v}");
}
```

## Error Representation { #error-representation }

In the example above, we used a simple string describing the problem as the error. Of course, this approach is extremely inconvenient for programmatic error handling. It is much more efficient to store error information in an enum, where each variant represents a specific problem.

As an example, let's write a function that takes a string containing a full name and returns the middle name.

```rust
#[derive(Debug)]
enum NameParseError{
    EmptyString,  // Attempting to parse an empty string
    NoMiddleName, // The string does not contain a middle name
}

// Takes a string containing a first name, middle name, and last name, 
// separated by spaces, and returns either the middle name or an error.
fn get_middle_name(full_name: &str) -> Result<String, NameParseError> {
    if full_name.is_empty() {
        return Err(NameParseError::EmptyString);
    }
    let mut words = split_to_words(full_name);
    if words.len() < 3 {
        return Err(NameParseError::NoMiddleName);
    }
    let middle_name = words.remove(1);
    Ok(middle_name)
}

// This function splits a string into words.
// The standard library has built-in functionality for this,
// but we aren't ready to use it just yet.
fn split_to_words(text: &str) -> Vec<String> {
    let mut words: Vec<String> = Vec::new();
    let mut current_word = String::new();
    for c in text.chars() {
        if c.is_whitespace() {
            if !current_word.is_empty() {
                words.push(current_word);
                current_word = String::new();
            }
        } else {
            current_word.push(c);
        }
    }
    if !current_word.is_empty() {
        words.push(current_word);
    }
    words
}

fn main() {
    let m_name_1 = get_middle_name("");
    println!("{m_name_1:?}"); // Err(EmptyString)

    let m_name_2 = get_middle_name("John Doe");
    println!("{m_name_2:?}"); // Err(NoMiddleName)

    let m_name_3 = get_middle_name("Sergei Petrovich Ivanov");
    println!("{m_name_3:?}"); // Ok("Petrovich")
}
```

## The Error Trait { #trait-error }

As mentioned, the `Result` type allows you to use any type to represent an error.
However, the standard library provides the [std::error::Error](https://doc.rust-lang.org/std/error/trait.Error.html) trait, which is specifically designed for types used as errors in `Result`:

```rust
pub trait Error: Debug + Display {
    // If this error is a wrapper for another error,
    // this method returns the wrapped error.
    fn source(&self) -> Option<&(dyn Error + 'static)> { ... }

    // Deprecated, use fmt::Debug instead
    fn description(&self) -> &str { ... }

    // Deprecated, use source
    fn cause(&self) -> Option<&dyn Error> { ... }

    // Available only in the Rust nightly build
    fn provide<'a>(&'a self, request: &mut Request<'a>) { ... }
}
```

This trait is used for error unification. All error types from the Rust standard library API implement `std::error::Error`.

Let's implement the Error trait for the error type from the [Error Representation](#error-representation) section.

```rust
#[derive(Debug)]
enum NameParseError{
    EmptyString,
    NoMiddleName,
}

impl std::fmt::Display for NameParseError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            NameParseError::EmptyString =>
                write!(f, "Attempt to parse empty string"),
            NameParseError::NoMiddleName =>
                write!(f, "No middle name found"),
        }
    }
}

impl std::error::Error for NameParseError {}
```

Once this is done, our error can be chained with other errors from the standard library.

> [!NOTE]
> Manually implementing the `std::error::Error` trait like this is rare, as there are libraries in the Rust ecosystem that significantly simplify this process. We will discuss them later.

## Composing Result Objects

Let's look at a simple program that prints a text file to the console.

> [!NOTE]
> МWe haven't covered the file system API yet and will do so in the [File System](../dive-deeper/file-system.md) chapter. However, if you have ever worked with a file system in other programming languages, this code should look familiar:

```rust,noplayground
use std::fs::File;
use std::io::{Error, prelude::*};

/// A function that reads the contents of a text file with a given name
/// and returns the file contents as a String object.
fn read_text_file(file_name: &str) -> Result<String, Error> {
    // Opening the file
    let mut file = match File::open(file_name) {
        Ok(file) => file,
        Err(e)   => return Err(e),
    };

    // Creating a buffer string where the content will be read
    let mut contents = String::new();
    // Reading the file contents into the string
    match file.read_to_string(&mut contents) {
        Ok(read_bytes) => Ok(contents),
        Err(e)         => return Err(e),
    }
} 

fn main() {
    // The /etc/fstab file is present on every Linux system.
    // If you are on Windows, replace it with the path to any text file.
    match read_text_file("/etc/fstab") {
        Ok(txt) => println!("{}", txt),
        Err(e)  => println!("Failed, because {}", e)
    }
}
```

It is easy to see that in the `read_text_file` function, there is as much boilerplate code for propagating the error as there is "useful" code. Can we improve the situation?

The `Result` type, like the `Option` type, has `map` and `and_then` combinators. Using them, we can rewrite the `read_text_file` function like this:

```rust,noplayground
use std::fs::File;
use std::io::{Error, prelude::*};

fn read_text_file(file_name: &str) -> Result<String, Error> {
    File::open(file_name).and_then(|mut file| {
        let mut contents = String::new();
        file.read_to_string(&mut contents).map(|_| {
            contents
        })
    })
}

fn main() {
    match read_text_file("/etc/fstab") {
        Ok(txt) => println!("{}", txt),
        Err(e)  => println!("Failed, because {}", e)
    }
}
```

This version of `read_text_file` is significantly shorter, but it has lost some readability.

Fortunately, Rust provides a special `?` operator, which allows you to maintain linear code flow while getting rid of boilerplate error propagation. This operator works as follows:

* If the `Result` object contains `Ok(value)`, the value is simply extracted.
* If the `Result` object contains `Err(error)`, the function terminates and the error is returned from the function.

```rust
use std::fs::File;
use std::io::{Error, prelude::*};

fn read_text_file(file_name: &str) -> Result<String, Error> {
    let mut file = File::open(file_name)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
} 

fn main() {
    match read_text_file("/etc/fstab") {
        Ok(txt) => println!("{}", txt),
        Err(e)  => println!("Failed, because {}", e)
    }
}
```

As you can see, thanks to the `?` operator, the function's code has become completely linear: as if there were no `Result` wrapper at all. At the same time, no error goes unnoticed. It is thanks to this powerful combination of the `Result` type, the `Error` trait, and the `?` operator that working with errors in Rust is convenient, despite the absence of an exception mechanism.

Naturally, the `?` operator can only be used inside functions that themselves return a `Result`.

## Ignoring Result

If we call a function that returns a `Result` and do not assign its result to a variable or check the `Result` for an error, the compiler will issue a warning, assuming we simply forgot to handle a possible error.

If we intentionally want to ignore the result, we should explicitly discard it by assigning it to a ["discarded" variable](variables.md#discarded-variable) — `let _ =` .

For example:

```rust
fn function_that_may_fail() -> Result<(), String> {
    Err("Something is wrong".to_string())
}

fn main() {
    let _ = function_that_may_fail();
}
```


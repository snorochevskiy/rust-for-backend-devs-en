# File System

> [!NOTE]
> Working with the file system is rarely needed when writing backend applications, so we will cover this topic very briefly.

For working with the file system, the Rust standard library provides the following modules:

* [std::fs](https://doc.rust-lang.org/std/fs/index.html) — contains functionality for working directly with file system objects: files, directories, and links.
* [std::io](https://doc.rust-lang.org/std/io/index.html) — contains functionality for I/O (Input/Output) operations.

## Reading and Writing Files

As an example of working with files, let's write a simple program that:

* Creates a new file, writes text to it, and closes the file.
* Opens the same file, appends a line to it, and closes the file.
* Opens the file again and reads its entire content into a string.

```rust
use std::fs::{File, OpenOptions};
use std::io::{self, Read, Write};

fn main() -> io::Result<()> {
    {
        // Creates a new file (or overwrites an existing one)
        let mut file = File::create("file.txt")?;
        // Writes bytes to the file
        file.write_all("First line\n".as_bytes())?;
        file.flush()?; // Очистка буфера вывода
        file.write_all("Second line\n".as_bytes())?;
    }

    {
        // Open the file for appending
        let mut file = OpenOptions::new()
            .append(true)
            .create(false)
            .open("file.txt")?;
        file.write_all("Third line\n".as_bytes())?;
    }

    {
        // Open the file for reading
        let mut file = File::open("file.txt")?;
        let mut buffer = String::new();
        file.read_to_string(&mut buffer)?;
        println!("{buffer}");
    }

    Ok(())
}
```

The first thing you might notice is that we open the file for writing but never explicitly close it. This is because the `Drop` trait is implemented for the `File` type: when the object goes out of scope, the destructor flushes the output buffer (by calling `flush`) and then closes the file.

The `io::Result<T>` type, which wraps the result of all I/O methods, is a type alias defined as:

```rust,noplyground
pub type Result<T> = result::Result<T, std::io::Error>;
```

In other words, it is a standard `Result` parameterized with the standard error type for I/O operations.

## Reading a Directory

Let's write a program that prints the names of files and directories located in the current directory:

```rust
use std::{ffi::OsString, fs::FileType};

fn main() -> std::io::Result<()> {
    for read_entry in std::fs::read_dir(".")? {
        if let Ok(entry) = read_entry {
            let entry_name: OsString = entry.file_name();
            let file_type: FileType = entry.file_type()?;
            println!("{entry_name:?} {file_type:?}");
        }
    }
    Ok(())
}
```

Program output:

```
"target" FileType { is_file: false, is_dir: true, is_symlink: false, .. }
"Cargo.lock" FileType { is_file: true, is_dir: false, is_symlink: false, .. }
"src" FileType { is_file: false, is_dir: true, is_symlink: false, .. }
"Cargo.toml" FileType { is_file: true, is_dir: false, is_symlink: false, .. }
```

As you can see, the directory name is stored in an object of type OsString. This is an important detail that deserves a closer look.

## OsString

Almost all functions for working with the file system use [OsString](https://doc.rust-lang.org/std/ffi/struct.OsString.html) instead of `String`. This type stores strings in the representation used by the current operating system.

A separate string type is used because:

* In Unix-like systems, filenames are stored as a sequence of non-zero bytes, which may contain UTF-8 or another encoding.
* On Windows, filenames are represented as sequences of non-zero 16-bit values (UTF-16).
* Rust strings are always stored in UTF-8 encoding, and null characters are allowed.

`OsString` implements `From<String>` and `From<&str>`, allowing you to easily convert a "regular" string into an `OsString`.

```rust
use std::ffi::OsString;

fn main() {
    let string = "text";
    let os_string = OsString::from(string);
}
```

Just as the `String` type has a corresponding string slice `&str`, `OsString` has a corresponding slice type — `&OsStr`.

```rust
use std::ffi::{OsStr, OsString};

fn main() {
    let string = "text";
    let os_string = OsString::from(string);
    let os_str: &OsStr = &os_string;
}
```

***

Most functions that handle paths and filenames use the [Path](https://doc.rust-lang.org/std/path/struct.Path.html) type. In fact, this type is essentially just a wrapper around `OsStr`:

```rust
pub struct Path {
    inner: OsStr,
}
```

The standard library provides `From`/`Into` and `AsRef` conversions that allow for seamless conversion from Rust strings to `Path`.

For example, the signature of the `File::create` method we used earlier looks like this:

```rust
pub fn create<P: AsRef<Path>>(path: P) -> io::Result<File>
```

Because `AsRef<Path>` is implemented for `&str`, we were able to call the method like this:

```rust
let mut file = File::create("file.txt")?;
```


# Input/Output

The Rust standard library provides the [std::io](https://doc.rust-lang.org/std/io/index.html) module, which contains universal interfaces and types for I/O operations.

This module features two core traits that define a unified interface for reading and writing: [Read](https://doc.rust-lang.org/std/io/trait.Read.html) and [Write](https://doc.rust-lang.org/std/io/trait.Write.html).

## Read

The `Read` trait defines one mandatory method `read` and several helper methods built on top of it.

```rust
pub trait Read {
    fn read(&mut self, buf: &mut [u8]) -> Result;
    // ...
}
```

Calling `read` fills the provided slice with bytes and returns the number of bytes written to the slice. If the slice size is smaller than the number of available bytes, `read` must be called multiple times.

The remaining methods of the `Read` trait have default implementations. Some include:

* `fn read_to_end(&mut self, buf: &mut Vec<u8>) -> Result<usize>`\
  Reads all content into a byte vector.
* `fn read_to_string(&mut self, buf: &mut String) -> Result<usize>`\
  Reads all content into a string. It assumes the data is text.
* `fn read_buf(&mut self, buf: BorrowedCursor<'_>) -> Result<()>`\
  Reads content into a cursor (discussed later).

The `Read` trait is implemented for a variety of types: files, network sockets, Unix pipes, STDIN (standard input), and more.

***

Let's look at how to use the `Read` trait with the `VecDeque<u8>` collection, which we studied in the previous chapter and which also implements `Read`.

> [!NOTE]
> Java programmers can draw an analogy to reading from a byte array using the `java.io.ByteArrayInputStream` class.

```rust,edition2024
use std::{collections::VecDeque, io::Read};

fn main() {
    // Create a VecDeque to read from via the Read trait
    let mut v: VecDeque<u8> = VecDeque::from(vec![1, 2, 3, 4, 5, 6, 7, 8, 9]);

    // A 2-byte buffer to read into
    let mut buf: [u8; 2] = [0; 2];

    // Read the first bytes
    let read_bytes: Result<usize, std::io::Error> = v.read(&mut buf);
    println!("Buffer: {buf:?}, Number of read bytes: {read_bytes:?}");

    // Read the remaining bytes in a loop
    while let Ok(read_bytes) = v.read(&mut buf) && read_bytes > 0 {
        println!("Buffer: {buf:?}, Number of read bytes: {read_bytes:?}");
    }
}
```

Program output:

```
Buffer: [1, 2], Number of read bytes: Ok(2)
Buffer: [3, 4], Number of read bytes: 2
Buffer: [5, 6], Number of read bytes: 2
Buffer: [7, 8], Number of read bytes: 2
Buffer: [9, 8], Number of read bytes: 1
```

If the data source is small (all data fits easily in RAM), it is more convenient to use the `read_to_end` method:

```rust
use std::{collections::VecDeque, io::Read};

fn main() {
    let mut v = VecDeque::from(vec![1, 2, 3, 4, 5, 6, 7, 8, 9]);

    let mut buf: Vec<u8> = Vec::new();
    let read_bytes: Result<usize, std::io::Error> = v.read_to_end(&mut buf);
    println!("Buffer: {buf:?}, Number of read bytes: {read_bytes:?}");
    // Buffer: [1, 2, 3, 4, 5, 6, 7, 8, 9], Number of read bytes: Ok(9)
}
```

If the data being read is text, the `read_to_string` method is suitable:

```rust
use std::{collections::VecDeque, io::Read};

fn main() {
    // 65 - 'A', 66 - 'B', 67 - 'C', 68 - 'D', 69 - 'E'
    let mut v = VecDeque::from(vec![65, 66, 67, 68, 69]);

    let mut buf = String::new();
    let read_bytes: Result<usize, std::io::Error> = v.read_to_string(&mut buf);
    println!("Buffer: {buf}, Number of read bytes: {read_bytes:?}");
    // Buffer: ABCDE, Number of read bytes: Ok(5)
}
```

## Cursor

While `VecDeque<u8>` implements `Read`, `Vec<u8>` does not. This is because `VecDeque` allows for efficient removal of elements from the front (bytes read from `VecDeque` via `read` are removed), whereas `Vec` can only remove elements efficiently from the end. However, if you need to work with `Vec<u8>` as a `Read` source, you can use the [std::io::Cursor](https://doc.rust-lang.org/std/io/struct.Cursor.html) wrapper.

`Cursor` has two fields: the wrapped collection and the current cursor position.

```rust
pub struct Cursor<T> {
    inner: T,
    pos: u64,
}
```

This position allows for an efficient implementation of the read operation: instead of removing bytes from the vector, the cursor simply shifts the position to the next unread bytes. This is exactly how `Read` is implemented for `Cursor`.

```rust
impl<T> Read for Cursor<T> where T: AsRef<[u8]> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        let n = Read::read(&mut Cursor::split(self).1, buf)?;
        self.pos += n as u64;
        Ok(n)
    }
}
```

As we can see, `Read` is implemented for any `Cursor` variant that wraps a type implementing `AsRef<[u8]>`. The `Vec<u8>` type implements `AsRef<[u8]>`.

```rust
use std::{io::Read, io::Cursor};

fn main() {
    let v: Vec<u8> = vec![65, 66, 67, 68, 69];
    let mut c: Cursor<Vec<u8>> = Cursor::new(v);
    let mut buf = String::new();
    let read_bytes: Result<usize, std::io::Error> = c.read_to_string(&mut buf);
    println!("String: {buf}, Number of read bytes: {read_bytes:?}");
}
```

Program output:

```
String: ABCDE, Number of read bytes: Ok(5)
```

## Write

The `Write` trait defines a universal interface for writing data.

The trait declares two mandatory methods and several others with default implementations.

```rust
pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;
    // ...
}
```

The `write` method writes bytes from the argument buffer and returns the number of bytes actually written. Specific implementations are allowed to write only part of the buffer if that part can be written quickly, while writing the rest would take significantly longer.

The `flush` method is relevant for types that buffer data in memory before the actual write. Calling this method should force the buffers to be flushed to the final storage.

The trait also contains a method with a default implementation: `write_all`.

```rust
fn write_all(&mut self, buf: &[u8]) -> Result<()>;
```

Unlike `write`, this method must write all bytes, even if it takes significantly more time.

Just like `Read`, the `Write` trait is implemented for files, network sockets, Unix pipes, STDOUT, and more.

***

Unlike `Read`, the `Write` trait is implemented for both `VecDeque<u8>` and `Vec<u8>`, as vectors allow for efficient addition of elements to the end.

Let's look at an example of writing bytes to a `Vec<u8>` using the `Write` trait:

```rust
use std::io::Write;

fn main() {
    let mut v: Vec<u8> = vec![1, 2, 3, 4, 5];
    let count = v.write(&[6, 7, 8, 9]);
    println!("Written: {count:?}, vec: {v:?}");
    // Written: Ok(4), vec: [1, 2, 3, 4, 5, 6, 7, 8, 9]
}
```

Standard output (STDOUT) also allows interaction through the `Write` trait:

```rust
use std::io::Write;

fn main() {
    let mut stdout = std::io::stdout();
    let _ = stdout.write(&[65, 66, 67, 68, 69, 10]);
}
```

The program will print:

```
ABCDE
```

***

In the next chapter, we will look at working with `Read` and `Write` using the file system as an example.

# Newtype Pattern

One of the most restrictive limitations in Rust is the Orphan Rule, which we previously mentioned in the [Traits](../rust-basics/traits.md#impl-for-foreign-types) chapter. To recap, the Orphan Rule states:

> A trait can be implemented for a type only if either the trait or the type (or both) belongs to the crate (library or program) where the implementation is defined.

In other words, if you want to implement trait A for type B, the implementation code must reside either in the crate where type B is declared or in the crate where trait A is declared.

Now, imagine a task where we need to sort a vector of file objects:

```rust,compile_fail
use std::fs::File;

fn main() {
    let mut v = vec![
        File::open("/etc/fstab").unwrap(),
        File::open("/etc/resolv.conf").unwrap(),
        File::open("/etc/hosts").unwrap(),
    ];
    v.sort();
}
```

> [!NOTE]
> For users unfamiliar with Linux:
> 
> * `/etc/fstab` — A standard configuration file for disk partitions.
> * `/etc/resolv.conf` — A file containing DNS server addresses.
> * `/etc/hosts` — A file used to map hostnames to IP addresses.
> 
> These files were chosen arbitrarily. When testing these examples, feel free to use any files available on your system or create new ones.

The `.sort()` method requires the type of the sorted objects to implement the [Ord](https://doc.rust-lang.org/std/cmp/trait.Ord.html) trait. The [std::fs::File](https://doc.rust-lang.org/std/fs/struct.File.html) type does not implement `Ord`, so compilation will fail with an error:

```
error[E0277]: the trait bound `File: Ord` is not satisfied
   --> src/main.rs:8:7
    |
  8 |     v.sort(); // the trait `Ord` is not implemented for `File`
    |       ^^^^ the trait `Ord` is not implemented for `File`
```

Suppose we want to implement `Ord` for `File` so that files are sorted based on their size. This is where the Orphan Rule becomes an issue: both the `File` type and the `Ord` trait are defined in the standard library, not in our crate.

The standard solution to this problem is the **Newtype pattern**. The idea is to wrap the "foreign" type in a tuple struct. Since this wrapper struct is defined in our crate, we own it.

```rust
struct Wrapper(ForeignType);
```

Now we can implement `Ord` for this wrapper. Let's do that and write a program to sort files by their size in ascending order.

```rust
use std::{cmp::Ordering, fs::File};

// Newtype wrapper for File.
struct FileWrapper(File);

impl PartialEq for FileWrapper {
    fn eq(&self, other: &Self) -> bool {
        match (self.0.metadata(), other.0.metadata()) {
            (Ok(m1), Ok(m2)) => m1.len() == m2.len(),
            _ => false,
        }
    }
}
impl Eq for FileWrapper {}

impl Ord for FileWrapper {
    fn cmp(&self, other: &Self) -> Ordering {
        match (self.0.metadata(), other.0.metadata()) {
            (Ok(m1), Ok(m2)) => m1.len().cmp(&m2.len()),
            _ => Ordering::Equal,
        }
    }
}

impl PartialOrd for FileWrapper {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

fn main() {
    let mut v: Vec<FileWrapper> = vec![
        FileWrapper(File::open("/etc/fstab").unwrap()),
        FileWrapper(File::open("/etc/resolv.conf").unwrap()),
        FileWrapper(File::open("/etc/hosts").unwrap()),
    ];

    println!("Before sorting");
    for file in v.iter() {
        println!("Size: {}", file.0.metadata().unwrap().len());
    }

    v.sort();

    println!("After sorting");
    for file in v.iter() {
        println!("Size: {}", file.0.metadata().unwrap().len());
    }
}
```

Running the program:

```
$ cargo run
Before sorting
Size: 866
Size: 920
Size: 219
After sorting
Size: 219
Size: 866
Size: 920
```

Everything works. However, as you may have noticed, we have to explicitly wrap `File` objects into `FileWrapper`. Furthermore, to access the internal file object, we have to use the `.0` index (e.g., `file.0.metadata()`), which isn't very elegant.

To solve this, when using the Newtype pattern, it is common practice to implement the `From` and `Deref` traits for the wrapper type. This allows you to:

* Wrap the original type by calling the `.into()` method.
* Access the wrapped object's methods directly without using the `.0` field.

Here is the improved version:

```rust
use std::{cmp::Ordering, fs::File, ops::Deref};

struct FileWrapper(File);

impl PartialEq for FileWrapper {
    fn eq(&self, other: &Self) -> bool {
        match (self.0.metadata(), other.0.metadata()) {
            (Ok(m1), Ok(m2)) => m1.len() == m2.len(),
            _ => false,
        }
    }
}
impl Eq for FileWrapper {}

impl Ord for FileWrapper {
    fn cmp(&self, other: &Self) -> Ordering {
        match (self.0.metadata(), other.0.metadata()) {
            (Ok(m1), Ok(m2)) => m1.len().cmp(&m2.len()),
            _ => Ordering::Equal,
        }
    }
}

impl PartialOrd for FileWrapper {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl From<File> for FileWrapper {
    fn from(value: File) -> Self {
        FileWrapper(value)
    }
}

impl Deref for FileWrapper {
    type Target = File;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

fn main() {
    let mut v: Vec<FileWrapper> = vec![
        // Convert File to the Newtype wrapper using .into()
        File::open("/etc/fstab").unwrap().into(),
        File::open("/etc/resolv.conf").unwrap().into(),
        File::open("/etc/hosts").unwrap().into(),
    ];

    println!("Before sorting");
    for file in v.iter() {
        // We can call file.metadata() instead of file.0.metadata()
        // because we implemented Deref
        println!("Size: {}", file.metadata().unwrap().len());
    }

    v.sort();

    println!("After sorting");
    for file in v.iter() {
        println!("Size: {}", file.metadata().unwrap().len());
    }
}
```

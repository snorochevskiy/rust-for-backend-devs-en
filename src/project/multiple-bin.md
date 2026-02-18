# Multiple Executable Binaries

As we know, when we create an executable program, the primary file is `src/main.rs`. But what if we want to create several executable files that reuse the same modules?

In such cases, we can create as many additional main files as we like in the `src/bin` directory.

For example, let's say we need three utilities working with the Fibonacci sequence:

```
0 1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987
```

* The first utility will print the first 10 elements of the sequence.
* The second will print the sum of the first 10 elements.
* The third will print the product of elements from the 2nd to the 10th (since multiplying by 0 always results in 0).

First, create a new Cargo project:

```
cargo new fibonacci_util
```

We won't have any dependencies, so we can leave `Cargo.toml` untouched for now.

Create the `src/fibonacci.rs` module, which implements the Fibonacci sequence generation as an iterator:

```rust
// How many elements of the Fibonacci sequence we want to take
pub struct FibonacciSequence(pub usize);

// Iterator for a Fibonacci sequence of a given length
pub struct FibonacciIter {
    current_index: usize,
    len: usize,
    prev: u64,
    before_prev: u64,
}

impl Iterator for FibonacciIter {
    type Item = u64;

    fn next(&mut self) -> Option<Self::Item> {
        match self.current_index {
            0 => {
                self.current_index += 1;
                Some(0)
            },
            1 => {
                self.current_index += 1;
                Some(1)
            },
            n if n < self.len => {
                let nth = self.before_prev + self.prev;
                self.before_prev = self.prev;
                self.prev = nth;
                self.current_index += 1;
                Some(nth)
            },
            _ => None,
        }
    }
}

impl IntoIterator for FibonacciSequence {
    type Item = u64;
    type IntoIter = FibonacciIter;

    fn into_iter(self) -> Self::IntoIter {
        FibonacciIter {
            current_index: 0,
            len: self.0,
            prev: 1,
            before_prev: 0,
        }
    }
}
```

Since this functionality will be shared by three programs, let's organize it as a library. Create `src/lib.rs`:

```rust
pub mod fibonacci;
```

At this point, your file tree should look like this:

```
fibonacci_util/
├── Cargo.toml
└── src/
    ├── fibonacci.rs
    ├── lib.rs
    └── main.rs
```

---

Now it’s time for the first program, which simply prints 10 elements of the sequence. Place it in `src/main.rs`:

```rust,noplayground
use fibonacci_util::fibonacci::FibonacciSequence;

fn main() {
    let s = FibonacciSequence(10).into_iter().collect::<Vec<_>>();
    println!("First 10 elements of fibonacci sequence: {s:?}");
}
```

Here we see something new: instead of connecting modules via mod, we simply import the module using `use`. The first component of the path is the name of the project itself (`fibonacci_util`). This is possible because we have a `lib.rs` file, which is connected by default under the crate's name. By referring to `fibonacci_util`, we are accessing our library in `lib.rs`.

Could we have just used `mod` in `main.rs` instead?

```rust,noplayground
mod fibonacci;
use fibonacci::FibonacciSequence;

fn main() {
    let s = FibonacciSequence(10).into_iter().collect::<Vec<_>>();
    println!("First 10 elements of fibonacci sequence: {s:?}");
}
```

Yes, we could. That code works too. However, this approach won't work for the next two utilities.

In a Cargo project, there can be only one primary entry point — `src/main.rs`. Only that file can include modules via `mod` from the `src/` directory. Additional binary files are placed in `/src/bin`, and they cannot directly access modules located in `src/`. They can only access them through `src/lib.rs`.

---

Create our second utility, which prints the sum of the first 10 elements, in `src/bin/fibonacci_sum.rs`:

```rust,noplayground
use fibonacci_util::fibonacci::FibonacciSequence;

fn main() {
    let sum: u64 = FibonacciSequence(10)
        .into_iter()
        .sum();
    println!("Sum of first 10 elements of fibonacci sequence: {sum}");
}
```

As you can see, the library functionality from `src/lib.rs` is available through the fibonacci_util namespace exactly as it was in `src/main.rs`. As we have said, none of main files in `src/bin` have direct access to modules in `src/` and the only way access these modules is via lib.rs`.

---

Finally, create the utility that prints the product of elements from the 2nd to the 10th — `src/bin/fibonacci_prod.rs`:

```rust,noplayground
use fibonacci_util::fibonacci::FibonacciSequence;

fn main() {
    let prod: u64 = FibonacciSequence(10).into_iter()
        .skip(1) // Skip the first element (zero)
        .product();
    println!("Product of fibonacci sequence elements from 2nd to 10th: {prod}");
}
```

By now, your file tree should look like this:

```
fibonacci_util/
├── Cargo.toml
└── src/
    ├── bin/
    │   ├── fibonacci_sum.rs
    │   └── fibonacci_prod.rs
    ├── fibonacci.rs
    ├── lib.rs
    └── main.rs
```

Running `cargo build` will produce the following in `target/debug`:

* `libfibonacci_util.rlib` — The Rust library compiled from `src/lib.rs`
* `fibonacci_util` — The executable compiled from `src/main.rs`
* `fibonacci_sum` — The executable compiled from `src/bin/fibonacci_sum.rs`
* `fibonacci_prod` — The executable compiled from `src/bin/fibonacci_prod.rs`

If you want to build only a specific binary, use the `--bin` flag: E.g. to build only `fibonacci_sum` execute:

```
cargo build --bin fibonacci_sum
```

## [[bin]]

By default, the executables will have the same names as their corresponding `src/bin/*.rs` files. However, you can change these names using the `[[bin]]` section in `Cargo.toml`.

Suppose we want `src/bin/fibonacci_sum.rs` to compile into an executable named `sum_10` and `src/bin/fibonacci_prod.rs` into `prod_10`.

```toml
[package]
name = "fibonacci_util"
version = "0.1.0"
edition = "2024"

[[bin]]
name = "sum_10"
path = "src/bin/fibonacci_sum.rs"

[[bin]]
name = "prod_10"
path = "src/bin/fibonacci_prod.rs"

[dependencies]
```

After running `cargo build`, you will find `sum_10` and `prod_10` in the `target/debug` directory. You can also rename the primary binary from `src/main.rs` in this way:

```toml
[package]
name = "fibonacci_util"
version = "0.1.0"
edition = "2024"

[[bin]]
name = "sequence_10"
path = "src/main.rs"

[[bin]]
name = "sum_10"
path = "src/bin/fibonacci_sum.rs"

[[bin]]
name = "prod_10"
path = "src/bin/fibonacci_prod.rs"

[dependencies]
```


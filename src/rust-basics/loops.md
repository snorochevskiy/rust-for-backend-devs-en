# Loops

## while

The while loop in Rust works and looks the same as in other imperative languages:

```rust,noplayground
while condition {
  // loop body
}
```

For example, printing numbers from `5` down to `0` (exclusive) to the console:

```rust
fn main() {
    let mut n = 5;
    while n > 0 {
        println!("{n}");
        n -= 1;
    }
}
```


## do-while

Rust does not have a do-while loop, but if absolutely necessary, it can be simulated using `while`:

<table class="two-side">
<tr>
<td>
This code

```rust
fn main() {
    let mut start = 0;
    let mut sum = 0;

    while {
        sum += start;
        start += 1;
        start < 10
    } {};

    println!("sum: {}", sum);
}
```

</td>
<td>
works like

```rust
fn main() {
    let mut start = 0;
    let mut sum = 0;

    do {
        sum += start;
        start += 1;
    } while start < 10;

    println!("sum: {}", sum);
}
```

</td>
</tr>
</table>


## loop

Rust has a special "infinite" loop — `loop`. Essentially, it is just an analog of `while true`.

```rust,noplayground
loop {
  loop body
}
```

Just like with a `while true` loop, you can exit a loop using the `break` statement.

For example: a program that prints Fibonacci sequence elements that are less than a specified number (assuming the number is always greater than 1).

```rust
fn main() {
    let maximum = 30;
    let mut a = 0;
    let mut b = 1;
    print!("{a} {b}");
    loop {
        let next = a + b;
        if next > maximum {
            break;
        }
        print!(" {next}");
        a = b;
        b = next;
    }
}
```

The `break` statement for the `loop` cycle has another feature: it returns a value.

```rust
let mut counter = 0;
let result = loop {
    counter += 1;
    if counter == 10 {
        break counter * 2; // returning from the loop
    }
};
println!("The result is {}", result); // 20
```

The type of the `break` statement is the _never type_.

## for

Rust lacks the "classic" **for** loop (like in C) — it only has **for-each**, which is intended for iterating over elements of sequences.

Синтаксис:

```rust,noplayground
for variable in sequence {
    // loop body
}
```

Example: iterating over array elements.

```rust
fn main() {
    let arr = [10, 20, 30, 40, 50];
    
    for element in arr {
        println!("the value is: {}", element);
    }
}
```

Using `for`, you can iterate over arrays, slices, vectors, and a number of other collections.

## Iterating over a Range

Rust doesn't have a "classic" for loop of the form for `(int i=0; i<N; i++)`. However, iterating over a numerical range is required quite often.

Fortunately, Rust has **Ranges**, which are defined as start `..` end and can be used for iteration in a `for` loop.

```rust
for i in 0 .. 10 {
    print!("{}, ", i);
}
// Prints: 0, 1, 2, 3, 4, 5, 6, 7, 8, 9,
```

This loop iterates through numbers from `0` up to `10` (exclusive).

To iterate from `0` to `10` (inclusive), you should use `0 ..= 10` instead of `0 .. 10`.

> [!NOTE]
> We will discuss the for loop in more detail in the [Ownership](ownership.md) and [Iterators](iterators.md) chapters.
> We will also take a closer look at ranges in the [Iterators](iterators.md)chapter.


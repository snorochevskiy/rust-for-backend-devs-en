# Destructuring

In the [Tuples](tuples.md) chapter, we were introduced to a special syntax for "breaking down" a tuple object into its constituent parts.

```rust
let employee: (&str, i32, bool) = ("John Doe", 1980, true);
let (name, birth_year, is_active_employee) = employee;
```

This operation of "breaking down" an object into its components is called **destructuring assignment**.

Destructuring assignment is available not only for tuples but also for arrays and structs. Let’s look at a few examples.

## Tuples

While we are already familiar with tuple destructuring, let's look at a more complex example involving a nested tuple.

```rust
let tup: (i32,char,bool,(i32,i32,i32)) = (1, 'z', true, (7,7,7));
let (num, c, _, t) = tup;
println!("num={}, char={}, triplet={:?}", num, c, t);
```

In the example above, we assigned the entire nested tuple to the variable `t`. However, we can also destructure that nested tuple directly.

```rust
let tup: (i32,char,bool,(i32,i32,i32)) = (1, 'z', true, (7,8,9));
let (num, c, _, (d1, d2, d3)) = tup;
println!("num={num}, char={c}, d1={d1}, d2= {d2}, d3={d3}");
```

Notice that for elements we aren't interested in, we used the underscore `_` as a "discarded" variable.

## Arrays

Arrays can also be destructured.

```rust
let arr: [i32;3] = [1, 2, 3];
let [a1, a2, a3] = arr;
println!("a1={a1}, a2={a2}, a3={a3}");
```

When destructuring an array, you can assign a portion of the elements from the beginning to variables and ignore the remaining elements at the end ("the tail").

```rust
let arr: [i32;5] = [1, 2, 3, 4, 5];
let [a_1, _, a_3, ..] = arr;
println!("a1={}, a3={}", a_1, a_3);
```

You can also capture those remaining elements into a new array instead of just ignoring them:

```rust
let arr: [i32;5] = [1, 2, 3, 4, 5];
let [a_1, _, a_3, rest @ ..] = arr;
// Тип переменной rest - массив [i32, 2]
println!("a1={}, a3={}, rest={:?}", a_1, a_3, rest);
```

The expression `rest@..` means: bind all remaining elements to the variable `rest`.

We cannot perform destructuring assignment for slices or vectors because the compiler cannot guarantee that the number of variables in the destructuring pattern matches the number of elements in the slice. An array, unlike a slice, has a fixed size known at compile time.

## Structs

Just as tuples are destructured based on the position of their elements, structs are destructured based on the names of their fields.

```rust
struct Person { name: String, age: u32 }

fn main() {
    let p = Person { name: String::from("John"), age: 25 };
    let Person { name, age } = p;
    println!("Name={}, Age={}", name, age);
}
```

When destructuring a struct, we can also "extract" only the specific fields we need:

```rust
struct Person { name: String, age: u32 }

fn main() {
    let p = Person { name: String::from("John"), age: 25 };
    let Person { name, .. } = p;
    println!("Name={name}");
}
```

## Destructuring Arguments

Destructuring isn't limited to assignment statements; it can also be used when receiving arguments in a function.

```rust
struct Point2D {
    x: i32,
    y: i32,
}

fn print_point_2d(Point2D { x, y }: Point2D) {
    println!("2D point ({x}, {y})");
}

fn main() {
    let p = Point2D { x: 1, y: 5 };
    print_point_2d(p);
}
```

Naturally, you can also destructure arguments that are tuple structs.

```rust
// Red, Green, Blue, Alpha
struct Rgba(u8, u8, u8, u8);

// Checks if the color is NOT semi-transparent,
// i.e., the alpha component equals 255
fn is_fully_opaque(Rgba(_, _, _, alpha): Rgba) -> bool {
    alpha == 255
}

fn main() {
    println!("Is opaque: {}", is_fully_opaque(Rgba(125, 0, 0, 255)));
    println!("Is opaque: {}", is_fully_opaque(Rgba(200, 200, 200, 50)));
}
```

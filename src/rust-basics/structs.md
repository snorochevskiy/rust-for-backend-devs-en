# Structures

In Rust, structures are a primary example of composite types.

A **structure** is a named collection of fields of various types that forms a new data type.

The syntax for declaring a structure is as follows:

```rust,noplayground
struct StructName {
    field_1: Type1,
    field_2: Type2,
    ...
    field_N: TypeN,
}
```

Note that you can place a comma after the last field of the structure, similar to how commas are used after the last argument in a function declaration.

Like variables, structure fields must be named using snake_case. Structures names must follow PascalCase (the name starts with a capital letter, and if there are multiple words, each word also starts with a capital letter), such as `User`, `MainAddress`, or `DatabaseConnection`.

Example of a structure storing a person's first and last name:

```rust
struct Person {
    first_name: String,
    last_name: String,
}
```

The syntax for creating an instance of a structure is:

```rust,noplayground
let variable = StructName { field_1: value_1, ..., field_N: value_N };
```

Например:

```rust,noplayground
let person = Person {
    first_name: String::from("John"),
    last_name: String::from("Doe"),
};
```

Accessing a structure field is done using the dot operator: `struct.field`.

```
my_struct.field_1
```

Example:

```rust
struct Person {
    first_name: String,
    last_name: String,
}

fn get_full_name(p: &Person) -> String {
    format!("{} {}", p.first_name, p.last_name)
}

fn main() {
    let p = Person {
        first_name: "John".to_string(),
        last_name: "Doe".to_string()
    };
    let full_name = get_full_name(&p);
    println!("{}", full_name); // "John Doe"
}
```

---

If you initialize a field with a variable that has the same name as the field, you can omit the variable name (e.g., `first_name` instead of `first_name: first_name`).

```rust
let first_name = String::from("John");
let person = Person {
    first_name,
    last_name: String::from("Doe"),
};
```

---

To change field values, the entire variable must be declared as mutable using `mut`.

```rust
fn main() {
    let mut p = Person {
        first_name: "John".to_string(),
        last_name: "Doe".to_string()
    };
    p.first_name = "Theodor".to_string();
}
```

## Creating One Instance from Another

Rust provides a specific syntax for creating a new object from an existing one by changing only specific fields:

```rust,noplayground
let new_object = Structure {
    field1: new_value_1,
    field2: new_value_2,
    ..old_object
};
```

E.g.:

```rust
struct Person {
    first_name: String,
    last_name: String,
}

fn main() {
    let p1 = Person {
        first_name: "John".to_string(),
        last_name: "Doe".to_string()
    };

    let p2 = Person { first_name: "Robert".to_string(), ..p1};

    println!("{} {}", p2.first_name, p2.last_name); // Robert Doe
}
```

## Methods

Unlike traditional OOP languages where methods are declared within the class body, Rust methods are declared separately from the structure using an impl block.

Syntax:

```rust,noplayground
impl ИмяСтруктуры {

  // метод, который НЕ меняет вызывающий объект
  fn метод_1(&self, аргумент_1: Тип1, …, аргумент_N: ТипN) -> ТипРезультата {
    ...
  }

  // метод, который меняет вызывающий объект
  fn метод_2(&mut self, аргумент_1: Тип1, …, аргумент_N: ТипN) -> ТипРезультата {
    ...
  }

  // "статический" метод, вызываемый на структуре, а не на объекте структуры
  fn метод_3(аргумент_1: Тип1, …, аргумент_N: ТипN) -> ТипРезультата {
    ...
  }
}
```

The equivalent of the `this` keyword in C++ or Java is `self`, which must be explicitly passed as the first argument:

* `&self`: Used if the method does not change the state of the object (immutable reference).
* `&mut self`: Used if the method needs to modify at least one field (mutable reference).
* `self`: Used in rare cases where the method takes ownership of the object.

Example:

```rust
struct Person {
  first_name: String,
  last_name: String,
  age: u32
}

impl Person {
  fn new(first: &str, last: &str) -> Person {
    Person {
      first_name: first.to_string(),
      last_name: last.to_string(),
      age: 0
    }
  }

  fn change_age(&mut self, new_age: u32) {
    self.age = new_age;
  }

  fn introduce(&self) -> String {
    format!("{} {} is {} years old", self.first_name, self.last_name, self.age)
  }
}

fn main() {
  let mut p = Person::new("John", "Doe");
  p.change_age(25);
  println!("{}", p.introduce());
}
```

Associated functions (methods without `self`) act as "static" methods called on the type itself (e.g., `Person::new()` from the example above).

## Tuple Structs

Rust also offers **tuple structures**, where fields are identified by position rather than name.

The syntax:

```rust,noplayground
struct Name(Type1, Type2, ..., TypeN);
```

In practice, a tuple structure is a tuple with a meaningful name and the ability to have methods attached to it. Fields are accessed using indexes like `.0` or `.1`.

Usge example:

```rust
/// Represent color encoded with RGB channels
struct RGB (u8, u8, u8);

impl RGB {
    /// Packs all the 3 channels into on 4-bytes number
    fn as_u32(&self) -> u32 {
        ((self.0 as u32) << 16)
            + ((self.1 as u32) << 8)
            + (self.2 as u32)
    }
}

fn main() {
    let mut color: RGB = RGB(255, 0, 0);  // red color
    println!("Red channel: {}", color.0); // Red channel: 255

    color.1 = 255; // Set green channel to 255

    // Tuple struct can be destructured inti components same as regular tuple
    let RGB(r, g, b) = color;

    println!("R={r}, G={g}, B={b}"); // R=255, G=255, B=0

    println!("As number: {}", color.as_u32());
}
```

## Unit-like Structs (Singleton)

You can create a structure with no fields:

```rust
struct Universe;
```

Obviously, there can be only one possible instance of such type, that is why this type of structures is called **unit-like** or **singleton** structure. An object of singleton structure behaves in the same way as an object of a regular structure with fields.

```rust
struct Universe;

impl Universe {
    fn includes(&self, p: &Planet) -> bool {
        true
    }
}

struct Planet {
    name: String
}

fn main() {
    let universe = Universe;

    let earth = Planet { name: "Earth".to_string() };
    println!("{}", universe.includes(&earth)); // true
}
```

These are useful when you need to implement a trait on a type but don't need to store any data within the type itself. We will learn about traints later in the [Traits](traits.md) chapter.

## Lifetimes in Structures

In the [Lifetime](lifetimes.md) chapter, we learned what lifetimes are in general and how to define them for functions. Now, let’s explore how lifetimes work with structs.

If a struct contains a field that stores a reference, you must specify a lifetime for that reference. This allows the compiler to ensure that the data being referenced lives at least as long as the struct itself.

Consider an example: we have a `String` containing a first and last name separated by a space. We want to create a struct with two fields:

* A reference to the part of the string containing the first name.
* A reference to the part of the string containing the last name.

```rust
#[derive(Debug)]
struct NameComponents<'a> {
    first_name: &'a str,
    last_name: &'a str,
}

fn main() {
    let full_name = "John Doe".to_string();

    let space_position = full_name.find(" ").unwrap();

    let components = NameComponents {
        first_name: &full_name[0..space_position],
        last_name: &full_name[space_position + 1 ..],
    };
    println!("{components:?}");
    // NameComponents { first_name: "John", last_name: "Doe" }
}
```

Thanks to the lifetime annotation, the compiler can verify that the `NameComponents` instance does not "outlive" the string its fields refer to. For example, the following code will not compile:

```rust,compile_fail
fn main() {
    let components;
    {
        let full_name = "John Doe".to_string();

        let space_position = full_name.find(" ").unwrap();

        // Error: `full_name` does not live long enough
        components = NameComponents {
            first_name: &full_name[0..space_position],
            last_name: &full_name[space_position + 1 ..],
        };
    }
    println!("{components:?}");
}
```

## Memory Layout

> [!WARNING]
> If you aren't familiar with computer architecture or how data is organized on the stack, this section might feel a bit dense. If so, don't worry about it too much—this info isn't strictly necessary for everyday Rust backend development. However, it’s a great topic to circle back to later!

Now, let's look at how struct instances are laid out in RAM.

> [!NOTE]
> The results in the examples below were received on an x86_64 processor using Rust version 1.92.

Let's use the standard [size_of](https://doc.rust-lang.org/std/mem/fn.size_of.html) function, which returns the size of a type in bytes.

```rust
struct MyStruct {
    a: i64,
    b: i32, 
}

fn main() {
    println!("Size = {}", std::mem::size_of::<MyStruct>()); // Size = 16
}
```

Wait, if the struct consists of an 8-byte field and a 4-byte field, why does it occupy 16 bytes instead of 12?

The reason is **alignment**. The compiler arranges fields so that the CPU can address them in a single read or write operation.

> [!NOTE]
> Processors in the x86_64 family have 8-byte registers. During a read operation, the CPU can only address memory blocks whose addresses are multiples of 8 (e.g., 0–7, 8–15, 16–23). A CPU cannot directly request a range like 4–11.
> 
> If a value starts at an address that isn't a multiple of 8 (like 4), the CPU has to read two adjacent blocks (0–7 and 8–15), use bitwise operations to "cut out" the desired parts, and then "glue" them together. These extra steps waste clock cycles. To avoid this, it’s more efficient to align values in memory, even if it means wasting a little space. These gaps in memory are called **padding**.

Let's see the specific addresses where fields `a` and `b` are located. To do this, we’ll get a pointer to each field and use the [addr()](https://doc.rust-lang.org/std/primitive.pointer.html#method.addr) method to get the numeric address.

```rust
struct MyStruct {
    a: i32,
    b: i64, 
}

fn main() {
    println!("Size = {}", std::mem::size_of::<MyStruct>()); // Size = 16

    let s = MyStruct { a: 1, b: 2 };
    println!("a: {}", ((&s.a) as *const i32).addr()); // a: 140731421349072
    println!("b: {}", ((&s.b) as *const i64).addr()); // b: 140731421349064
}
```

Both addresses `140731421349072` and `140731421349064` are multiples of 8.

> [!NOTE]
> Usually, we use the `{:p}` format specifier in `println!` to print a pointer, but here we wanted to show the raw numeric address explicitly.

Now, let's look at how arrays of structs are laid out:

```rust
struct MyStruct {
    a: i32,
    b: i64, 
}

fn main() {
    let arr = [
        MyStruct { a: 1, b: 2 },
        MyStruct { a: 3, b: 4 },
        MyStruct { a: 5, b: 6 }
    ];
    println!("arr[0].a: {:p}", &arr[0].a); // arr[0].a: 0x7ffdc124d970
    println!("arr[0].b: {:p}", &arr[0].b); // arr[0].b: 0x7ffdc124d968
    println!("arr[1].a: {:p}", &arr[1].a); // arr[1].a: 0x7ffdc124d980
    println!("arr[1].b: {:p}", &arr[1].b); // arr[1].b: 0x7ffdc124d978
    println!("arr[2].a: {:p}", &arr[2].a); // arr[2].a: 0x7ffdc124d990
    println!("arr[2].b: {:p}", &arr[2].b); // arr[2].b: 0x7ffdc124d988
}
```

In memory, it looks like this:

![](img/struct_array_layout.svg)

Here, you can clearly see both the alignment and the padding.

You might also notice that the compiler reordered fields `a` and `b`. Rust does not guarantee that fields in memory will follow the same order they appear in your code. The compiler may reorder them to minimize padding and save space.

If you need the compiler to maintain the exact order of fields (for example, when interacting with libraries written in C), you should use the `#[repr(C)]` attribute.

```rust
#[repr(C)]
struct MyStruct {
    a: i32,
    b: i64, 
}
```

This annotation tells Rust to use a memory representation compatible with the C language, where field order is preserved exactly as declared.

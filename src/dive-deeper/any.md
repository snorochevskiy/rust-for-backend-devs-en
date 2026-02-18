# Any

## The Downcasting Problem

As we know, Rust is not an Object-Oriented Programming (OOP) language. However, thanks to its trait system, Rust offers powerful polymorphic capabilities.

Using [trait objects](../rust-basics/traits.md#dynamic_dispatching), we can "upcast" at any time, i.e. start interacting with a type through its trait by abstracting away the specific concrete type hidden behind the trait object.

```rust
trait Person {}

struct Student {}
impl Person for Student {}

struct Teacher {}
impl Person for Teacher {}

fn work_with_person(person: &dyn Person) {}

fn main() {
    let child1 = Student {};
    work_with_base(&child1); // upcast &Student to &dyn Person
    
    let child2 = Teacher {};
    work_with_base(&child2); // upcast &Teacher to &dyn Person
}
```

But what do we do when we need to find out which specific type is hidden behind a trait object? Referring back to our example: how can we take a `&dyn Person` reference and obtain a `&Student` or `&Teacher` reference from it? In other words, how do we "downcast"?


> [!NOTE]
> "Upcasting" and "downcasting" are terms from the OOP world, where classes can inherit from one another. Converting a pointer of a child class to a pointer of a parent class is called **upcasting** (moving "up" the hierarchy). Conversely, converting a pointer from a parent class to a child class is called **downcasting**.
> 
> For example, assume we have following hierarchy of classes in C++:
> 
> ```cpp
> class Shape {
> protected:
>    float x, y; // location
> }
> 
> class Circle : public Shape {
>     float radius;
> }
> ```
> 
> Тогда
> 
> ```cpp
> Circle* circle1 = new Circle();
> Shape*  shape   = dynamic_cast<Shape*>(circle1); // upcast
> Circle* circle2 = dynamic_cast<Circle*>(shape);  // downcast
> ```

Unfortunately, Rust currently provides no built-in way to determine the concrete type hidden behind a trait object. Remember, a reference to a trait object is simply a pair of pointers (a "fat pointer"): the first points to the object itself, and the second points to the <ins>vtable</ins>. The vtable does not contain the type's name or ID; it only contains function pointers for methods, a pointer to the destructor (if any), and information about size and alignment.

However, there is a workaround. During compilation, every type is assigned a unique identifier — [TypeId](https://doc.rust-lang.org/std/any/struct.TypeId.html), which is essentially a wrapper around a 128-bit number:

```rust
pub struct TypeId {
    t: (u64, u64),
}
```

To find the `TypeId` for a specific type, we use the [TypeId::of](https://doc.rust-lang.org/std/any/struct.TypeId.html#method.of) constructor.

Example:

```rust
use std::any::TypeId;

struct Student {}
struct Teacher {}

fn main() {
    println!("{:?}", TypeId::of::<Student>()); // TypeId(0xaf8ebb053b28606d84daaf35f1eae84c)
    println!("{:?}", TypeId::of::<Teacher>()); // TypeId(0x318ec7010fde8de82dedcdc9562881fd)
}
```

It is important to note that `TypeId::of` is evaluated at compile time.

Armed with `TypeId`, we can manually implement a check: is this trait object an instance of a specific type or not? Let’s rewrite the first example to include downcasting.

```rust
use std::any::TypeId;

trait Person where Self: 'static {
    fn exact_type(&self) -> TypeId {
        TypeId::of::<Self>()
    }
}

/// Note: the impl is for 'dyn Person' (the trait object), not 'Person'
impl dyn Person {
    /// This method downcasts the trait object to a reference of a concrete type
    /// if that type matches the object hidden behind the trait object.
    fn downcast<T: 'static>(&self) -> Option<&T> {
        if TypeId::of::<T>() == self.exact_type() {
            unsafe {
                let (data, _vtable): (*const u8, *const u8) =
                    std::mem::transmute(self);
                let data: *const T = std::mem::transmute(data);
                data.as_ref()
            }
        } else {
            None
        }
    }
}

struct Student {
    name: String,
    year_of_education: u32,
}
impl Person for Student {}

struct Teacher {
    name: String,
    subject: String,
}
impl Person for Teacher {}

fn work_with_person(base: &dyn Person) {
    // Use our downcast method to check who is hidden behind the trait object
    if let Some(s) = base.downcast::<Student>() {
        println!("This is {}, a {}-year student", s.name, s.year_of_education);
    } else if let Some(t) = base.downcast::<Teacher>() {
        println!("This is {}, a teacher of {}", t.name, t.subject);
    }
}

fn main() {
    let student = Student {
        name: "John".to_string(),
        year_of_education: 3,
    };
    work_with_person(&student);

    let teacher = Teacher {
        name: "Ivan".to_string(),
        subject: "Programming".to_string(),
    };
    work_with_person(&teacher);
}
```

The program works as expected:

```
$ cargo run
This is John, a 3-year student
This is Ivan, a teacher of Programming
```

Breaking Down the Code

For the `Person` trait, we added a method that returns the `TypeId` of the type implementing the trait. If `Person` is implemented by `Student`, it returns the ID for `Student`; if by `Teacher`, it returns the ID for `Teacher`.

```rust
trait Person where Self: 'static {
    fn exact_type(&self) -> TypeId {
        TypeId::of::<Self>()
    }
}
```

> [!TIP]
> <details>
> 
> <summary>Regarding the <code>'static</code> trait bound</summary>
> 
> The `'static` bound indicates that the <ins>object</ins> implementing this trait must not contain any references with a lifetime shorter than `'static`.
> 
> Consider this example:
> 
> ```rust
> struct MyIntRef<'a> {
>     r: &'a i32,
> }
> 
> /// Function that accepts and object with 'static lifetime
> fn prove_static<T: 'static>(v: T) {}
> 
> fn main() {
>     // Это работает
>     static VAR_STATIC: i32 = 5;
>     let my1 = MyIntRef { r: &VAR_STATIC };
>     prove_static(my1);
> 
>     // This will not compile
>     let var_local = 5;
>     let my2 = MyIntRef { r: &var_local }; // doesn not live long enough
>     prove_static(my2);
> }
> ```
> 
> Does this mean our downcast from `Person` to `Student` / `Teacher` won't work if we add a struct that contains non-`'static` references? Yes, exactly. Unfortunately, this is a current limitation. In practice, however, you will rarely need to downcast a type containing non-`static` references.
> 
> </details>

Next, specifically for the trait object `dyn Person`, we defined a method that checks if the `TypeId` of the target type (the one we want to downcast to) matches the `TypeId` of the hidden object. If it matches, we extract the data pointer from the trait object and cast it to the desired reference type. Otherwise, we return `None`.

```rust
impl dyn Base {
    fn downcast<T: 'static>(&self) -> Option<&T> {
        if TypeId::of::<T>() == self.exact_type() {
            unsafe {
                // Destructuring the reference to the trait object into a pair of pointers
                let (data, _vtable): (*const u8, *const u8) =
                    std::mem::transmute(self);
                // Casting the generic (*u8) pointer into a pointer to a data of the given type
                let data: *const T = std::mem::transmute(data);
                // Casting pointer to a reference: as_ref() returns Option
                data.as_ref()
            }
        } else {
            None
        }
    }
}
```

To understand how we get a reference to the hidden object, we must recall what a trait object reference looks like. It consists of two pointers:

* The first points directly to the data.
* The second points to the vtable.

Поэтому мы сначала преобразовываем `self` (ссылку на трэйт-объект) в пару указателей, затем берём первый указатель и приводим его к ссылке желаемого типа.

To convert `self` into a tuple of two pointers, we use [transmute](https://doc.rust-lang.org/std/mem/fn.transmute.html). This function allows us to view the same memory (the same sequence of bytes) through the lens of a different type. Naturally, this function must be used with extreme caution, accounting for size and alignment. Fortunately, in this case, we are dealing with two pointers, which always have a size equal to the machine word and do not require special alignment.

> [!TIP]
> Actually, the fragment:
> 
> ```rust
> let (data, _vtable): (*const u8, *const u8) = std::mem::transmute(self);
> let data: *const T = std::mem::transmute(data);
> data.as_ref()
> ```
> 
> could be written much more simply:
> 
> ```rust
> Some(&*(self as *const dyn Person as *const T));
> ```
> 
> However, such a notation hides the mechanical details of working with trait objects that we wanted to demonstrate explicitly.

The rest of the example code should be straightforward.

## Downcasting with Any

Of course, implementing such a downcast manually is inconvenient and dangerous. In the previous example, we only did it to demonstrate how downcasting works at the memory level.

In practice, the Rust standard library provides a (relatively) convenient trait wrapper, [Any](https://doc.rust-lang.org/std/any/trait.Any.html), which encapsulates TypeId, thereby enabling downcasting.

The `Any` trait looks like this:

```rust
pub trait Any: 'static {
    fn type_id(&self) -> TypeId;
}

impl<T: 'static + ?Sized> Any for T {
    fn type_id(&self) -> TypeId {
        TypeId::of::<T>()
    }
}
```

As seen from the declaration, an implementation of `Any` is defined for absolutely any type that satisfies the `'static` bound. This implementation simply retrieves the type's `TypeId`. Essentially, this is the same logic we used with the `exact_type` method in the previous example.

Additionally, the following methods are defined for `dyn Any`:

* `pub fn is<T: Any>(&self) -> bool`\
  Returns `true` if the actual type hidden behind the trait object is `T`.
* `pub fn downcast_ref<T: Any>(&self) -> Option<&T>`\
  If the actual type hidden behind the trait object is `T`, it returns `Some(&T)`. Otherwise, it returns `None`.
* `pub fn downcast_mut<T: Any>(&mut self) -> Option<&mut T>`\
  If the actual type hidden behind the trait object is `T`, it returns `Some(&mut T)`. Otherwise, it returns `None`.

Here is an example of a simple downcast using `Any`:

```rust
use std::any::Any;

fn main() {
    let s = "hello".to_string();
    let any = &s as &dyn Any;

    println!("any is i32: {}", any.is::<i32>());       // any is i32: false
    println!("any is String: {}", any.is::<String>()); // any is String: true

    println!("{:?}", any.downcast_ref::<i32>());    // None
    println!("{:?}", any.downcast_ref::<String>()); // Some("hello")
}
```

If a trait inherits from `Any`, its trait object automatically gains the ability to perform downcasting.

Let's rewrite our custom downcasting example using `Any`:

```rust
use std::any::Any;

trait Person: Any {}

struct Student {
    name: String,
    year_of_education: u32,
}
impl Person for Student {}

struct Teacher {
    name: String,
    subject: String,
}
impl Person for Teacher {}

fn work_with_person(person: &dyn Person) {
    let any = person as &dyn Any;
    if let Some(s) = any.downcast_ref::<Student>() {
        println!("This is {}, a {}-year student", s.name, s.year_of_education);
    } else if let Some(t) = any.downcast_ref::<Teacher>() {
        println!("This is {}, a teacher of {}", t.name, t.subject);
    }
}

fn main() {
    let student = Student {
        name: "John".to_string(),
        year_of_education: 3,
    };
    work_with_person(&student);

    let teacher = Teacher {
        name: "Ivan".to_string(),
        subject: "Programming".to_string(),
    };
    work_with_person(&teacher);
}
```

The output of the program is the same as with our manual downcast:

```
$ cargo run
This is John, a 3-year student
This is Ivan, a teacher of Programming
```


# Serialization

## serde

The Rust language does not have any built-in standard mechanisms for data serialization and deserialization. However, the [serde](https://crates.io/crates/serde) library has long been the de facto standard for this purpose.

serde does not contain the logic for serializing into specific formats (like JSON or XML). Instead, it provides traits and procedural macros that allow for the automatic generation of special methods for custom types. These methods provide meta-information about the type and allow for reading and writing its fields. Other libraries that work with specific formats (JSON, XML, YAML) then use these methods to perform the actual serialization and deserialization.

The serde crate provides two core traits: [Serialize](https://docs.rs/serde/latest/serde/trait.Serialize.html) and [Deserialize](https://docs.rs/serde/latest/serde/trait.Deserialize.html). The first is required for serialization, and the second for deserialization.

![](img/serde_impl.svg)

(The serde_json crate uses the type description provided by the `Serialize` trait implementation to serialize an object of `MyType` into JSON)

---

Suppose we have an `Employee` type that we want to be able to serialize into various formats.

```rust,noplayground
struct Employee {
  id: u64,
  name: String,
}
```

First, add the serde dependency to your `Cargo.toml`.

```toml
[package]
name = "test_rust"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = { version = "1", features = ["derive"] }
```

Next, we must annotate our structure with the `Serialize` and `Deserialize` traits.

```rust,noplayground
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
struct Employee {
  id: u64,
  name: String,
}
```

Now, `Serialize` and `Deserialize` implementations will be generated for the `Employee` struct, allowing serialization libraries for specific formats to interact with it.

> [!TIP]
> <details>
> 
> <summary>What exactly does serde generate?</summary>
> 
> To see the implementation code for `Deserialize` and `Serialize` for our `Employee` struct, let's create a simple program consisting only of the struct declaration.
> 
> ```rust,noplayground
> use serde::{Deserialize, Serialize};
> 
> #[derive(Serialize, Deserialize)]
> struct Employee {
>     id: u64,
>     name: String,
> }
> 
> fn main() {}
> ```
> 
> Next, run the `cargo expand` command, which prints the contents of `main.rs` to the console after all macros have been processed.
> 
> (If the `expand` utility is not installed, first run `cargo install cargo-expand`)
> 
> ```rust,noplayground
> #![feature(prelude_import)]
> #[macro_use]
> extern crate std;
> #[prelude_import]
> use std::prelude::rust_2024::*;
> use serde::{Deserialize, Serialize};
> struct Employee {
>     id: u64,
>     name: String,
> }
> #[doc(hidden)]
> #[allow(
>     non_upper_case_globals,
>     unused_attributes,
>     unused_qualifications,
>     clippy::absolute_paths,
> )]
> const _: () = {
>     #[allow(unused_extern_crates, clippy::useless_attribute)]
>     extern crate serde as _serde;
>     #[automatically_derived]
>     impl _serde::Serialize for Employee {
>         fn serialize<__S>(
>             &self,
>             __serializer: __S,
>         ) -> _serde::__private228::Result<__S::Ok, __S::Error>
>         where
>             __S: _serde::Serializer,
>         {
>             let mut __serde_state = _serde::Serializer::serialize_struct(
>                 __serializer,
>                 "Employee",
>                 false as usize + 1 + 1,
>             )?;
>             _serde::ser::SerializeStruct::serialize_field(
>                 &mut __serde_state,
>                 "id",
>                 &self.id,
>             )?;
>             _serde::ser::SerializeStruct::serialize_field(
>                 &mut __serde_state,
>                 "name",
>                 &self.name,
>             )?;
>             _serde::ser::SerializeStruct::end(__serde_state)
>         }
>     }
> };
> #[doc(hidden)]
> #[allow(
>     non_upper_case_globals,
>     unused_attributes,
>     unused_qualifications,
>     clippy::absolute_paths,
> )]
> const _: () = {
>     #[allow(unused_extern_crates, clippy::useless_attribute)]
>     extern crate serde as _serde;
>     #[automatically_derived]
>     impl<'de> _serde::Deserialize<'de> for Employee {
>         fn deserialize<__D>(
>             __deserializer: __D,
>         ) -> _serde::__private228::Result<Self, __D::Error>
>         where
>             __D: _serde::Deserializer<'de>,
>         {
>             #[allow(non_camel_case_types)]
>             #[doc(hidden)]
>             enum __Field {
>                 __field0,
>                 __field1,
>                 __ignore,
>             }
>             #[doc(hidden)]
>             struct __FieldVisitor;
>             #[automatically_derived]
>             impl<'de> _serde::de::Visitor<'de> for __FieldVisitor {
>                 type Value = __Field;
>                 fn expecting(
>                     &self,
>                     __formatter: &mut _serde::__private228::Formatter,
>                 ) -> _serde::__private228::fmt::Result {
>                     _serde::__private228::Formatter::write_str(
>                         __formatter,
>                         "field identifier",
>                     )
>                 }
>                 fn visit_u64<__E>(
>                     self,
>                     __value: u64,
>                 ) -> _serde::__private228::Result<Self::Value, __E>
>                 where
>                     __E: _serde::de::Error,
>                 {
>                     match __value {
>                         0u64 => _serde::__private228::Ok(__Field::__field0),
>                         1u64 => _serde::__private228::Ok(__Field::__field1),
>                         _ => _serde::__private228::Ok(__Field::__ignore),
>                     }
>                 }
>                 fn visit_str<__E>(
>                     self,
>                     __value: &str,
>                 ) -> _serde::__private228::Result<Self::Value, __E>
>                 where
>                     __E: _serde::de::Error,
>                 {
>                     match __value {
>                         "id" => _serde::__private228::Ok(__Field::__field0),
>                         "name" => _serde::__private228::Ok(__Field::__field1),
>                         _ => _serde::__private228::Ok(__Field::__ignore),
>                     }
>                 }
>                 fn visit_bytes<__E>(
>                     self,
>                     __value: &[u8],
>                 ) -> _serde::__private228::Result<Self::Value, __E>
>                 where
>                     __E: _serde::de::Error,
>                 {
>                     match __value {
>                         b"id" => _serde::__private228::Ok(__Field::__field0),
>                         b"name" => _serde::__private228::Ok(__Field::__field1),
>                         _ => _serde::__private228::Ok(__Field::__ignore),
>                     }
>                 }
>             }
>             #[automatically_derived]
>             impl<'de> _serde::Deserialize<'de> for __Field {
>                 #[inline]
>                 fn deserialize<__D>(
>                     __deserializer: __D,
>                 ) -> _serde::__private228::Result<Self, __D::Error>
>                 where
>                     __D: _serde::Deserializer<'de>,
>                 {
>                     _serde::Deserializer::deserialize_identifier(
>                         __deserializer,
>                         __FieldVisitor,
>                     )
>                 }
>             }
>             #[doc(hidden)]
>             struct __Visitor<'de> {
>                 marker: _serde::__private228::PhantomData<Employee>,
>                 lifetime: _serde::__private228::PhantomData<&'de ()>,
>             }
>             #[automatically_derived]
>             impl<'de> _serde::de::Visitor<'de> for __Visitor<'de> {
>                 type Value = Employee;
>                 fn expecting(
>                     &self,
>                     __formatter: &mut _serde::__private228::Formatter,
>                 ) -> _serde::__private228::fmt::Result {
>                     _serde::__private228::Formatter::write_str(
>                         __formatter,
>                         "struct Employee",
>                     )
>                 }
>                 #[inline]
>                 fn visit_seq<__A>(
>                     self,
>                     mut __seq: __A,
>                 ) -> _serde::__private228::Result<Self::Value, __A::Error>
>                 where
>                     __A: _serde::de::SeqAccess<'de>,
>                 {
>                     let __field0 = match _serde::de::SeqAccess::next_element::<
>                         u64,
>                     >(&mut __seq)? {
>                         _serde::__private228::Some(__value) => __value,
>                         _serde::__private228::None => {
>                             return _serde::__private228::Err(
>                                 _serde::de::Error::invalid_length(
>                                     0usize,
>                                     &"struct Employee with 2 elements",
>                                 ),
>                             );
>                         }
>                     };
>                     let __field1 = match _serde::de::SeqAccess::next_element::<
>                         String,
>                     >(&mut __seq)? {
>                         _serde::__private228::Some(__value) => __value,
>                         _serde::__private228::None => {
>                             return _serde::__private228::Err(
>                                 _serde::de::Error::invalid_length(
>                                     1usize,
>                                     &"struct Employee with 2 elements",
>                                 ),
>                             );
>                         }
>                     };
>                     _serde::__private228::Ok(Employee {
>                         id: __field0,
>                         name: __field1,
>                     })
>                 }
>                 #[inline]
>                 fn visit_map<__A>(
>                     self,
>                     mut __map: __A,
>                 ) -> _serde::__private228::Result<Self::Value, __A::Error>
>                 where
>                     __A: _serde::de::MapAccess<'de>,
>                 {
>                     let mut __field0: _serde::__private228::Option<u64> = _serde::__private228::None;
>                     let mut __field1: _serde::__private228::Option<String> = _serde::__private228::None;
>                     while let _serde::__private228::Some(__key) = _serde::de::MapAccess::next_key::<
>                         __Field,
>                     >(&mut __map)? {
>                         match __key {
>                             __Field::__field0 => {
>                                 if _serde::__private228::Option::is_some(&__field0) {
>                                     return _serde::__private228::Err(
>                                         <__A::Error as _serde::de::Error>::duplicate_field("id"),
>                                     );
>                                 }
>                                 __field0 = _serde::__private228::Some(
>                                     _serde::de::MapAccess::next_value::<u64>(&mut __map)?,
>                                 );
>                             }
>                             __Field::__field1 => {
>                                 if _serde::__private228::Option::is_some(&__field1) {
>                                     return _serde::__private228::Err(
>                                         <__A::Error as _serde::de::Error>::duplicate_field("name"),
>                                     );
>                                 }
>                                 __field1 = _serde::__private228::Some(
>                                     _serde::de::MapAccess::next_value::<String>(&mut __map)?,
>                                 );
>                             }
>                             _ => {
>                                 let _ = _serde::de::MapAccess::next_value::<
>                                     _serde::de::IgnoredAny,
>                                 >(&mut __map)?;
>                             }
>                         }
>                     }
>                     let __field0 = match __field0 {
>                         _serde::__private228::Some(__field0) => __field0,
>                         _serde::__private228::None => {
>                             _serde::__private228::de::missing_field("id")?
>                         }
>                     };
>                     let __field1 = match __field1 {
>                         _serde::__private228::Some(__field1) => __field1,
>                         _serde::__private228::None => {
>                             _serde::__private228::de::missing_field("name")?
>                         }
>                     };
>                     _serde::__private228::Ok(Employee {
>                         id: __field0,
>                         name: __field1,
>                     })
>                 }
>             }
>             #[doc(hidden)]
>             const FIELDS: &'static [&'static str] = &["id", "name"];
>             _serde::Deserializer::deserialize_struct(
>                 __deserializer,
>                 "Employee",
>                 FIELDS,
>                 __Visitor {
>                     marker: _serde::__private228::PhantomData::<Employee>,
>                     lifetime: _serde::__private228::PhantomData,
>                 },
>             )
>         }
>     }
> };
> fn main() {}
> ```
> 
> As you can see, the `Serialize` implementation is very simple. It consists of a method that provides metadata about the structure: the struct name and the list of its fields.
> 
> The implementation of `Deserialize` is much more complex. It generates a visitor (the Visitor pattern), which is used to traverse the fields of the structure.
> 
> </details>

## Сериализация в JSON

Now let's look at how to serialize our `Employee` struct into JSON.

First, we need to add a dependency for [serde_json](https://crates.io/crates/serde_json) in `Cargo.toml`:

```toml
[package]
name = "test_rust"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

Now we can use the following functions:

* [serde_json::to_string](https://docs.rs/serde_json/latest/serde_json/fn.to_string.html) — serializes an object into a JSON string.
* [serde_json::from_str](https://docs.rs/serde_json/latest/serde_json/fn.from_str.html) — deserializes a JSON string into an object.

Example:

```rust,edition2024
# use serde;
# use serde_json;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Employee {
    id: u64,
    name: String,
}

fn main() {
    // Create a struct instance
    let emp1 = Employee {
        id: 1,
        name: "John Doe".to_string(),
    };

    // Serialize to a JSON string
    let json = serde_json::to_string(&emp1).unwrap();
    println!("{json}"); // {"id":1,"name":"John Doe"}

    // Deserialize from a JSON string
    let emp2: Employee = serde_json::from_str(
        r#"{ "id" : 2, "name" : "Ivan Ivanov" }"#
    ).unwrap();
    println!("{emp2:?}"); // Employee { id: 2, name: "Ivan Ivanov" }
}
```

### serde_json::Value

The serde_json crate also defines the [Value](https://docs.rs/serde_json/latest/serde_json/enum.Value.html) enum, which is essentially a mapping of JSON types to a Rust enum.

```rust,noplayground
pub enum Value {
    Null,
    Bool(bool),
    Number(Number),
    String(String),
    Array(Vec<Value>),
    Object(Map<String, Value>),
}
```

If we want to parse a JSON document "manually", we can parse the document into a `Value` object, which can then be inspected in any way we find convenient.

To obtain a `Value` object from a JSON text document, we use the already familiar `serde_json::from_str` function:

```rust,noplayground
let v: Value = serde_json::from_str("{ \"a\": 1 }").expect("Invalid json");
```

After that, you can process the `Value` object using pattern matching.

Let's demonstrate this with an example:

```rust,edition2024
# use serde;
# use serde_json;
use serde_json::Value;

fn main() {
    // Parse a JSON document containing a number
    let v1: Value = serde_json::from_str("5").unwrap();
    match v1 {
        Value::Number(n) => println!("Number: {n}"),
        _ => (),
    }

    // Parse a JSON document containing a string
    let v2: Value = serde_json::from_str("\"text\"").unwrap();
    match v2 {
        Value::String(s) => println!("String: {s}"),
        _ => (),
    }

    // Parse a JSON document containing an array of numbers
    let v3: Value = serde_json::from_str("[1,2,3]").unwrap();
    match v3 {
        Value::Array(arr) => {
            print!("Array:");
            for e in arr {
                match e {
                    Value::Number(n) => print!(" {n}"),
                    _ => (),
                }
            }
            println!("");
        }
        _ => (),
    }

    // Parse a JSON document containing an object
    let v4: Value = serde_json::from_str(
        r#"
            {"id" : 1, "name" : "John Doe"}
        "#,
    )
    .unwrap();
    match v4 {
        Value::Object(fields) => {
            println!("Object:");
            match fields.get("id") {
                Some(Value::Number(n)) => println!("  id = {n}"),
                _ => (),
            };
            match fields.get("name") {
                Some(Value::String(s)) => println!("  name = {s}"),
                _ => (),
            };
        }
        _ => (),
    }
}
```

Program output:

```
Number: 5
String: text
Array: 1 2 3
Object:
  id = 1
  name = John Doe
```

## XML Serialization

Serialization to XML works on the same principle as serialization to JSON. Direct interaction with XML is performed using the[serde-xml-rs](https://crates.io/crates/serde-xml-rs) crate, which relies on `Serialize` and `Deserialize` implementations generated by Serde.

First, we need to add the serde-xml-rs dependency to `Cargo.toml`:

```toml
[package]
name = "test_rust"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde-xml-rs = "0.8.2"
```

Now we can adapt our JSON serialization example to work with XML:

```rust,noplayground
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Employee {
    id: u64,
    name: String,
}

fn main() {
    let emp1 = Employee {
        id: 1,
        name: "John Doe".to_string(),
    };
    let xml = serde_xml_rs::to_string(&emp1).unwrap(); // serialize to XML
    println!("{xml}");

    let emp2: Employee = serde_xml_rs::from_str( // deserialize from XML
        r#"<Employee><id>1</id><name>John Doe</name></Employee>"#
    ).unwrap();
    println!("{emp2:?}");
}
```

Program output:

```
$ cargo run
<?xml version="1.0" encoding="UTF-8"?><Employee><id>1</id><name>John Doe</name></Employee>
Employee { id: 1, name: "John Doe" }
```

As you can see, thanks to Serde, working with XML is very similar to working with JSON.

## Enum Serialization

Many data formats (such as JSON) do not natively support an "enum" type, which creates a slight challenge for serializing Rust objects.

Serde supports several enum serialization strategies, which we will examine through examples using the JSON format.

***

The default enum serialization strategy (Externally tagged) implies that an enum variant is serialized into a structure like this:

```
"variant_name": { value }
```

In this case, the enum object becomes the value of a field whose key is the name of the variant.

For example:

```rust,edition2024
# use serde;
# use serde_json;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
enum Employee {
    Programmer { name: String, language: String },
    Manager { name: String },
    OfficeCat(String),
}

fn main() {
    let programmer = Employee::Programmer {
        name: "John Doe".to_string(),
        language: "Rust".to_string(),
    };
    println!("{}", serde_json::to_string(&programmer).unwrap());
    // {"Programmer":{"name":"John Doe","language":"Rust"}}

    let manager = Employee::Manager {
        name: "Ivan Ivanov".to_string(),
    };
    println!("{}", serde_json::to_string(&manager).unwrap());
    // {"Manager":{"name":"Ivan Ivanov"}

    let cat = Employee::OfficeCat("Shadow".to_string());
    println!("{}", serde_json::to_string(&cat).unwrap());
    // {"OfficeCat":"Shadow"}
}
```

***

Another option for enum serialization is adding an additional "tag" field to the JSON object body to store the name of the variant. This is known as Internally tagged. The name of this tag field is defined using the `#[serde(tag = "name")]` annotation.

Let’s rewrite the previous example using a tag:

```rust,edition2024
# use serde;
# use serde_json;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type")]
enum Employee {
    Programmer { name: String, language: String },
    Manager { name: String },
}

fn main() {
    let programmer = Employee::Programmer {
        name: "John Doe".to_string(),
        language: "Rust".to_string(),
    };
    println!("{}", serde_json::to_string(&programmer).unwrap());
    // {"type":"Programmer","name":"John Doe","language":"Rust"}

    let manager = Employee::Manager {
        name: "Ivan Ivanov".to_string(),
    };
    println!("{}", serde_json::to_string(&manager).unwrap());
    // {"type":"Manager","name":"Ivan Ivanov"}
}
```

> [!IMPORTANT]
> Keep in mind that this strategy only works if all enum variants are "struct-like" (variants with named fields). Tuple variants are not supported in this mode.

***

If we want the enum's fields to be stored in a nested object rather than at the same level as the tag, we can specify a name for a field that will contain the serialized object body: `#[serde(tag = "name", content = "field")]`.


```rust,edition2024
# use serde;
# use serde_json;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type", content = "obj")]
enum Employee {
    Programmer { name: String, language: String },
    Manager { name: String },
    OfficeCat(String),
}

fn main() {
    let programmer = Employee::Programmer {
        name: "John Doe".to_string(),
        language: "Rust".to_string(),
    };
    println!("{}", serde_json::to_string(&programmer).unwrap());
    // {"type":"Programmer","obj":{"name":"John Doe","language":"Rust"}}

    let manager = Employee::Manager {
        name: "Ivan Ivanov".to_string(),
    };
    println!("{}", serde_json::to_string(&manager).unwrap());
    // {"type":"Manager","obj":{"name":"Ivan Ivanov"}}

    let cat = Employee::OfficeCat("Shadow".to_string());
    println!("{}", serde_json::to_string(&cat).unwrap());
    // {"type":"OfficeCat","obj":"Shadow"}
}
```

***

You can also use the `#[serde(untagged)]` annotation. This causes the enum to be serialized without any additional fields or tags identifying the variant. However, this should be used only as a last resort, as it can easily lead to ambiguity where correct deserialization becomes impossible.

```rust,edition2024
# use serde;
# use serde_json;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(untagged)]
enum Employee {
    Programmer { name: String, language: String },
    Manager { name: String },
}

fn main() {
    let programmer = Employee::Programmer {
        name: "John Doe".to_string(),
        language: "Rust".to_string(),
    };
    println!("{}", serde_json::to_string(&programmer).unwrap());
    // {"name":"John Doe","language":"Rust"}

    let manager = Employee::Manager {
        name: "Ivan Ivanov".to_string(),
    };
    println!("{}", serde_json::to_string(&manager).unwrap());
    // {"name":"Ivan Ivanov"}
}
```

## Renaming Fields

If we want the field name in the serialized format to differ from the name in the struct, we can use the `#[serde(rename = "serialized_name")]` annotation.

For example, suppose we want a field named `first_name` to be serialized as name.

```rust,edition2024
# use serde;
# use serde_json;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Employee {
    #[serde(rename = "name")]
    first_name: String,
}

fn main() {
    // deserialize
    let emp1: Employee = serde_json::from_str(r#"{"name":"John"}"#).unwrap();
    println!("{emp1:?}"); // Employee { first_name: "John" }

    // serialize
    let emp2 = Employee { first_name: "John Doe".to_string() };
    println!("{}", serde_json::to_string(&emp2).unwrap()); // {"name":"John Doe"}
}
```

## Naming Conventions

Field names in Rust follow snake_case: names start with a lowercase letter, and if a name consists of multiple words, they are separated by an underscore.

If you need the serialized field names to follow a different convention, you can specify this using the `#[serde(rename_all = "CONVENTION_NAME")]` annotation.

Available naming conventions:

* `lowercase` — namelikethis
* `UPPERCASE` — NAMELIKETHIS
* `PascalCase` — NameLikeThis
* `camelCase` — nameLikeThis
* `snake_case` — name_like_this
* `SCREAMING_SNAKE_CASE` — NAME_LIKE_THIS
* `kebab-case` — такое-имя-поля
* `SCREAMING-KEBAB-CASE` — NAME-LIKE-THIS

Example: Here, we specify that camelCase should be used during serialization and deserialization.

```rust,edition2024
# use serde;
# use serde_json;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
struct Employee {
    first_name: String,
}

fn main() {
    let emp: Employee = serde_json::from_str(r#"{"fistName":"John"}"#).unwrap();
    println!("{emp:?}");
}
```

## DeserializeOwned

Let’s take a closer look at the [Deserialize](https://docs.rs/serde/latest/serde/trait.Deserialize.html) trait:

```rust,noplayground
pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
       where D: Deserializer<'de>;
}
```

As you can see, it has a lifetime parameter `'de`, which is tied to the object from which the deserialization is performed.

In most situations, we don't even notice this lifetime parameter, but there are scenarios where it can become a problem. For instance, if we want to write a generic structure that stores a value of a deserializable type. We might try to write something like this:

```rust,noplayground
struct Holder<'de, T> where T: Deserialize<'de> {
    v: T,
}
```

However, the compiler will not allow this code to compile, throwing an error:

```
error[E0392]: lifetime parameter `'de` is never used
  | struct Holder<'de, T>
  |               ^^^ unused lifetime parameter
```

The issue is that the lifetime parameter is only used in the `Deserialize` trait bound, but it isn't used anywhere in the body of the structure.

Specifically to solve this problem, the Serde library provides a trait wrapper called [DeserializeOwned](https://docs.rs/serde/latest/serde/de/trait.DeserializeOwned.html):

```rust,noplayground
pub trait DeserializeOwned: for<'de> Deserialize<'de> { }
```

The declaration of `DeserializeOwned` uses HRTB (higher-ranked trait bounds) to move the lifetime constraint off the trait itself, thereby removing the mandatory trait parameter on the inheriting trait. When writing backend applications, this construction is relatively rare, so we won't dive into the technical details here.

Using `DeserializeOwned`, we can easily create a generic structure that stores a type implementing `Deserialize`.

```rust,edition2024
# use serde;
# use serde_json;
use serde::de::DeserializeOwned;

#[derive(Debug)]
struct Holder<T> where T: DeserializeOwned {
    v: T,
}

fn deserialize_into_holder<T: DeserializeOwned>(json: &str) -> Holder<T> {
    let v: T = serde_json::from_str(json).unwrap();
    Holder { v }
}

fn main() {
    let json = String::from("5");
    let h: Holder<i32> = deserialize_into_holder(&json);
    println!("{h:?}");
}
```

***

> [!TIP]
> Detailed documentation on Serde can be found on the official website: [https://serde.rs/](https://serde.rs/)

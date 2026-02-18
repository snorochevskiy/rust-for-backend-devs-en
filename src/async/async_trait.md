# async\_trait

In this chapter, we will explore a solution to a specific problem that arises when using async methods within traits.

Suppose we are developing a backend to handle certain products. We decided to organize the code into two components: one responsible for database interactions and another for business logic.

```rust
fn main() {
    let product_storage = ProductDbStorage {};
    let product_service = ProductService { product_storage };
    println!("All products: {:?}", product_service.get_all_products())
}

#[derive(Debug)]
struct Product { } // A product type

// Business logic for working with products
struct ProductService {
    product_storage: ProductDbStorage,
}
impl ProductService {
    fn get_all_products(&self) -> Vec<Product> {
        self.product_storage.list_products()
    }
}

// Database operations where products are stored
struct ProductDbStorage {
    // Encapsulates the DB connection
}
impl ProductDbStorage {
    fn list_products(&self) -> Vec<Product> {
        // Imagine a call to a real database happens here
        Vec::new()
    }
}
```

However, this tight coupling between components is inconvenient. For example, if we want to write a unit test for `ProductService`, we cannot initialize the `storage` field with anything other than `ProductDbStorage`, which relies on a real database. Since unit tests shouldn't depend on real external systems, these calls must be replaced with mocks.

To loosen the coupling between components, we can:

* Abstract the `ProductDbStorage` type using a `ProductStorage` trait.
* Store `Box<dyn ProductStorage>` in the `ProductService` struct instead of a concrete `ProductDbStorage` object.

With this structure, we can finally write a unit test.

```rust
fn main() {
    let product_storage = ProductDbStorage {};
    let product_service = ProductService {
        product_storage: Box::new(product_storage)
    };
    println!("All products: {:?}", product_service.get_all_products())
}

#[derive(Debug)]
struct Product { }

struct ProductService {
    product_storage: Box<dyn ProductStorage>,
}
impl ProductService {
    fn get_all_products(&self) -> Vec<Product> {
        self.product_storage.list_products()
    }
}

trait ProductStorage {
     fn list_products(&self) -> Vec<Product>;
} 

struct ProductDbStorage { }

impl ProductStorage for ProductDbStorage {
    fn list_products(&self) -> Vec<Product> { Vec::new() }
}

#[test]
fn test_product_service() {
    // Заглушка для ProductStorage
    struct ProductStorageMock;
    impl ProductStorage for ProductStorageMock {
        fn list_products(&self) -> Vec<Product> {
            vec![Product {}]
        }
    }

    let sut = ProductService {
        product_storage: Box::new(ProductStorageMock) // mock
    };

    assert_eq!(sut.get_all_products().len(), 1);
}
```

Now, let's try to rewrite `ProductService` and `ProductStorage` so that their methods are asynchronous.

```rust,compile_fail
struct ProductService {
    product_storage: Box<dyn ProductStorage>,
} //                 ^^^^^^^^^^^^^^^^^^^^^^^
  // Error: the trait `ProductStorage` is not dyn compatible
impl ProductService {
    async fn get_all_products(&self) -> Vec<Product> {
        self.product_storage.list_products().await
    }
}

trait ProductStorage {
    async fn list_products(&self) -> Vec<Product>;
} 

struct ProductDbStorage { }

impl ProductStorage for ProductDbStorage {
    async fn list_products(&self) -> Vec<Product> { Vec::new() }
}
```

Unfortunately, we will encounter a compilation error:

```
the trait ProductStorage is not dyn compatible
```

This error indicates that the compiler cannot create a trait object for a trait containing `async` methods. To understand why, we must remember that a function like:

```rust,noplayground
async fn myfunc() -> MyType {}
```

is transformed by the compiler into:

```rust,noplayground
fn myfunc() -> impl Future<Output = MyType> {}
```

This is where the problem lies. To create a trait object, the compiler must build a virtual call table (vtable). One requirement for this is that the compiler must know the size of the types returned by the methods (to allocate space on the stack), but this is impossible because `impl Future` has an unknown size.

The standard solution for this vtable issue is to wrap the return type in a `Box`, which has a known size on the stack. Since we are dealing with asynchronous code (and `Box` alone isn't sufficient for certain thread-safety requirements in this context), we should use `Pin<Box>`.

Armed with this knowledge, let's rewrite our example:

```rust,edition2024
use std::pin::Pin;

#[tokio::main]
async fn main() {
    let product_storage = ProductDbStorage {};
    let product_service = ProductService {
        product_storage: Box::new(product_storage)
    };
    println!("All products: {:?}", product_service.get_all_products().await)
}

#[derive(Debug)]
struct Product { }

struct ProductService {
    product_storage: Box<dyn ProductStorage + Send + Sync>,
}
impl ProductService {
    async fn get_all_products(&self) -> Vec<Product> {
        self.product_storage.list_products().await
    }
}

trait ProductStorage {
    fn list_products<'a, 't>(
        &'a self
    ) -> Pin<Box<dyn Future<Output = Vec<Product>> + Send + 't>>
    where 'a: 't, Self: 't;
} 

struct ProductDbStorage { }

impl ProductStorage for ProductDbStorage {
    fn list_products<'a, 't>(
        &'a self,
    ) -> Pin<Box<dyn Future<Output = Vec<Product>> + Send + 't>>
    where 'a: 't, Self: 't {
        Box::pin(async move { Vec::new() })
    }
}
```

This code compiles and runs successfully, but let's be honest: it doesn't look very elegant.

> [!NOTE]
> Note that instead of `Box<dyn ProductStorage>`, we are using `Box<dyn ProductStorage + Send + Sync>`

Fortunately, the [async-trait](https://crates.io/crates/async-trait) crate exists to simplify the situation. Just mark both the trait and its implementation with the `#[async_trait]` attribute, and you can write code using standard async methods without all the `Pin<Box>` boilerplate. During compilation, the attribute proc-macro will automatically replace the `async` methods with versions that return `Pin<Box>`.

```rust,edition2024
#[tokio::main]
async fn main() {
    let product_storage = ProductDbStorage {};
    let product_service = ProductService {
        product_storage: Box::new(product_storage)
    };
    println!("All products: {:?}", product_service.get_all_products().await)
}

#[derive(Debug)]
struct Product { }

struct ProductService {
    product_storage: Box<dyn ProductStorage + Send + Sync>,
}
impl ProductService {
    async fn get_all_products(&self) -> Vec<Product> {
        self.product_storage.list_products().await
    }
}

#[async_trait::async_trait]
trait ProductStorage {
     async fn list_products(&self) -> Vec<Product>;
} 

struct ProductDbStorage { }

#[async_trait::async_trait]
impl ProductStorage for ProductDbStorage {
    async fn list_products(&self) -> Vec<Product> { Vec::new() }
}
```


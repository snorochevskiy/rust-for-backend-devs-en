# Iterators

As we know, the `for` loop can be used to iterate over elements of various types: arrays, slices, vectors, ranges, and so on. But how is this flexibility achieved?

The reason is that the `for` loop only knows how to iterate over **iterators**, which, in turn, provide a unified interface for working with sequences. An iterator is represented by the [Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html) trait.

```rust
pub trait Iterator {
    type Item;
    
    fn next(&mut self) -> Option<Self::Item>;
    
    // ... over 70 other methods with default implementations
}
```

Effectively, an iterator defines an interface that allows for the sequential traversal of objects in a collection (array, slice, vector) with which the iterator is associated.

The most important method of this trait is `next`, which returns:

* Either the next element in the sequence being traversed in the form of `Some(значение)`
* Or `None` if all elements have been processed

It is the `next` method that the `for` loop uses to retrieve the next element.

> [!NOTE]
> It should be noted that <ins>usually</ins> the `Iterator` trait is not implemented for the collection type itself (array, vector, etc.), but for a separate type whose object is used to traverse the elements in the collection associated with the iterator.
> 
> ![](img/iterator.svg)

## Making an Iterator

To make it easier to understand, let's create our own iterator for a vector. The standard library already contains a standard iterator implementation for vectors—the [Iter](https://doc.rust-lang.org/beta/core/slice/struct.Iter.html) type. To avoid confusing the compiler about which iterator implementation it should use (the standard one or ours), we will create a wrapper around the vector.

```rust,noplayground
struct MyVec<T>(Vec<T>);
```

Now, let's create an iterator for our wrapper:

```rust,noplayground
struct MyVecIter<'a, T> {
    data: &'a MyVec<T>,
    current_ind: usize,
}

impl <'a, T> Iterator for MyVecIter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        if self.current_ind < self.data.0.len() {
            let current_element = &self.data.0[self.current_ind];
            self.current_ind += 1;
            Some(current_element)
        } else {
            None
        }
    }
}
```

Our iterator is simply a struct that stores a reference to a `MyVec` object and the index of the element to be retrieved during the next `next()` call.

If the index hasn't exceeded the boundaries of the vector being iterated, we return a reference to the element at that index and then increment the index. If the index is out of bounds, we return `None`, which serves as a signal that the iteration is complete.

Example of using our iterator:

```rust
# struct MyVec<T>(Vec<T>);
# 
# struct MyVecIter<'a, T> {
#     data: &'a MyVec<T>,
#     current_ind: usize,
# }
# 
# impl <'a, T> Iterator for MyVecIter<'a, T> {
#     type Item = &'a T;
# 
#     fn next(&mut self) -> Option<Self::Item> {
#         if self.current_ind < self.data.0.len() {
#             let current_element = &self.data.0[self.current_ind];
#             self.current_ind += 1;
#             Some(current_element)
#         } else {
#             None
#         }
#     }
# }
# 
fn main() {
    let my_vec = MyVec(vec![1,2,3]);
    let iterator = MyVecIter { data: &my_vec, current_ind: 0 }; 
    for n in iterator {
        print!("{n} ");
    }
    // Prints: 1 2 3
}
```

Of course, creating an iterator object manually like this is quite inconvenient. Therefore, collections that support iteration often include an `iter()` method that constructs the iterator itself.

```rust,noplayground
impl <T> MyVec<T> {
    // Returns an iterator for our MyVec wrapper
    fn iter(&self) -> MyVecIter<'_, T> {
        MyVecIter { data: self, current_ind: 0 }
    }
}
```

Now iterating with a `for` loop is much more convenient:

```rust
# struct MyVec<T>(Vec<T>);
# 
# impl <T> MyVec<T> {
#     fn iter(&self) -> MyVecIter<'_, T> {
#         MyVecIter { data: self, current_ind: 0 }
#     }
# }
# 
# struct MyVecIter<'a, T> {
#     data: &'a MyVec<T>,
#     current_ind: usize,
# }
# 
# impl <'a, T> Iterator for MyVecIter<'a, T> {
#     type Item = &'a T;
# 
#     fn next(&mut self) -> Option<Self::Item> {
#         if self.current_ind < self.data.0.len() {
#             let current_element = &self.data.0[self.current_ind];
#             self.current_ind += 1;
#             Some(current_element)
#         } else {
#             None
#         }
#     }
# }
# 
fn main() {
    let my_vec = MyVec(vec![1,2,3]);
    for n in my_vec.iter() {
        print!("{n}, ");
    }
}
```

> [!TIP]
> The method name `iter()` is not regulated by any trait; it is simply a clear, generally accepted convention.

## The for Loop and Iterators

Now that we've established that the `for` loop actually works only with iterators, let's look at how the `for` loop iterates directly over a vector.

It's simple: the `for` loop accepts either an iterator directly or a type that implements the [IntoIterator](https://doc.rust-lang.org/std/iter/trait.IntoIterator.html) trait.

```rust,noplayground
pub trait IntoIterator {
    // The type of the iterator's item
    type Item;

    // The type of the iterator
    type IntoIter: Iterator<Item = Self::Item>;

    // Returns the iterator object
    fn into_iter(self) -> Self::IntoIter;
}
```

This trait is implemented by the collection type itself and allows it to create an iterator object for its elements.

This means if we want to pass a `MyVec` object directly into a `for` loop, we must implement the `IntoIterator` trait for it. Let's do that.

```rust,noplayground
impl <'a, T> IntoIterator for &'a MyVec<T> {
    type Item = &'a T;

    type IntoIter = MyVecIter<'a, T>;

    fn into_iter(self) -> Self::IntoIter {
        MyVecIter { data: self, index: 0 }
    }
}
```

Since we are iterating over elements via reference rather than by value, we implement IntoIterator for `&MyVec` instead of `MyVec`.

Now that `IntoIterator` is defined, we can iterate over the `MyVec` object like this:

```rust
# struct MyVec<T>(Vec<T>);
# 
# struct MyVecIter<'a, T> {
#     data: &'a MyVec<T>,
#     index: usize,
# }
# 
# impl <T> MyVec<T> {
#     fn iter(&self) -> MyVecIter<'_, T> {
#         MyVecIter { data: self, index: 0 }
#     }
# }
# 
# impl <'a, T> Iterator for MyVecIter<'a, T> {
#     type Item = &'a T;
#     fn next(&mut self) -> Option<Self::Item> {
#         if self.index < self.data.0.len() {
#             let current_element = &self.data.0[self.index];
#             self.index += 1;
#             Some(current_element)
#         } else {
#             None
#         }
#     }
# }
# 
# impl <'a, T> IntoIterator for &'a MyVec<T> {
#     type Item = &'a T;
# 
#     type IntoIter = MyVecIter<'a, T>;
# 
#     fn into_iter(self) -> Self::IntoIter {
#         self.iter()
#     }
# }
# 
fn main() {
    let my_vec = MyVec(vec![1,2,3]);

    for n in &my_vec {
        print!("{n}, ");
    }
}
```

Is it possible to iterate over our wrapper's elements by value rather than by reference? Yes, but we would need to write a second iterator that captures our wrapper by value:

```rust
struct MyVec<T>(Vec<T>);

struct MyVecIterVal<T> {
    data: MyVec<T>,
}

impl <T> MyVec<T> {
    fn iter_val(self) -> MyVecIterVal<T> {
        MyVecIterVal { data: self }
    }
}

impl <T> Iterator for MyVecIterVal<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        self.data.0.pop() // extract and return the element
    }
}

impl <T> IntoIterator for MyVec<T> {
    type Item = T;

    type IntoIter = MyVecIterVal<T>;

    fn into_iter(self) -> Self::IntoIter {
        self.iter_val()
    }
}

fn main() {
    let my_vec = MyVec(vec![1,2,3]);

    for n in my_vec {
        print!("{n}, ");
    }
}
```

## Iterating Over a Vector - Revisited

Now that we have explored the `Iterator` and `IntoIterator traits`, let's look at several examples of iterating over types we are already familiar with.

```rust,noplayground
let v = vec![1, 2, 3];

// The iter() method is defined for the Vec type.
// It explicitly returns an iterator object that iterates over references to elements.
// Type of i: &i32
for i in v.iter() { }

// The into_iter() method will be implicitly called on the &Vec object,
// returning an iterator. Type of i: &i32
for i in &v { }

// Explicit call to into_iter() for the Vec object. This returns an iterator 
// that iterates over elements by value, meaning the iteration "consumes" the vector. 
// Type of i: i32
for i in v.into_iter() { }

// Implicitly calls into_iter() for the Vec object.
for i in v { }
```

The pattern is exactly the same for arrays:

```rust,noplayground
let arr = [1, 2, 3];

// Iteration by reference - i: &i32
for i in arr.iter() { }

// Iteration by reference - i: &i32
for i in &arr { }

// Iteration by value - i: i32
for i in arr.into_iter() { }

// Iteration by value - i: i32
for i in arr { }
```

To summarize:

* If we want to iterate over a collection by accessing elements by <ins>reference</ins>, we should pass either a reference to the collection to the `for` loop or call the `iter()` method on the collection object (if available).
* If we need the actual <ins>values</ins> of the elements (and we are okay with consuming/destroying the collection itself), we should pass the collection object directly to the `for` loop or call the `into_iter()` method.

## Range

In the [Loops](loops.md) chapter, we saw this form of numeric iteration:

```rust
for i in 0 .. 20 {
    println!("{i}");
}
```

Finally, we are ready to understand how this works under the hood.

The `..` operator creates a **range** — an object of the [Range](https://doc.rust-lang.org/std/ops/struct.Range.html) struct, which looks like this:

```rust,noplayground
pub struct Range<Idx> {
    pub start: Idx, // starting element of the range
    pub end: Idx,   // ending element of the range
}
```

As we can see, `Range` simply stores the start and end values. Furthermore, the `Range` type implements the `Iterator` interface, which allows us to iterate over its elements.

If desired, we can use `Range` without a `for` loop:

```rust
use std::ops::Range;

fn main() {
    let mut range: Range<i32> = 0..20;
    println!("{:?}", range.next()); // Some(0)
    println!("{:?}", range.next()); // Some(1)
}
```

> [!NOTE]
> At the beginning of the chapter, we mentioned that `Iterator` is usually implemented not for the collection itself, but for a separate type used to traverse the collection. The `Range` type is an exception, as the `Iterator` trait is implemented directly for it. In this case, iteration is implemented by simply incrementing the `start` field.

## Iterator API

As noted earlier, the `Iterator` trait defines over 70 methods in addition to `next()`, all of which have default implementations. What kind of methods are these?

They are methods that allow for the <ins>declarative</ins> processing and transformation of collections.

> [!TIP]
> Java programmers can draw an analogy with the Stream API, and C# programmers with LINQ.

Let's start with an example: we have a vector of numbers, and we want to:

1) Extract only the even numbers.
2) Square each of those even numbers.

```rust
let v1 = vec![1,2,3,4,5];
let v2 = v1.into_iter()     // Turn the vector into an iterator
    .filter(|x| x % 2 == 0) // Filter only even elements
    .map(|x| x * x)         // Square the elements
    .collect::<Vec<_>>();   // Repackage the iterator back into a vector
println!("{v2:?}"); // [4, 16]
```

Let's break this example down in detail:

1\) First, we obtain an iterator for the vector. Since we no longer need the original vector after processing, we use `into_iter()`. This creates an iterator that moves elements out of the vector by value, meaning the original vector is "consumed."

2\) We call the [filter](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter) method on the resulting iterator:

```rust
fn filter<P>(self, predicate: P) -> Filter<Self, P>
    where Self: Sized, P: FnMut(&Self::Item) -> bool
{
    Filter::new(self, predicate)
}
```

As we can see, this method takes a predicate and returns a [Filter](https://doc.rust-lang.org/std/iter/struct.Filter.html) object.

```rust
pub struct Filter<I, P> {
    pub(crate) iter: I,
    predicate: P,
}
```

The `Filter` type is a <ins>wrapper</ins> around an iterator; it stores the original iterator and the predicate used for filtering. Naturally, the `Filter` type also implements the `Iterator` trait.

3\) On the `Filter` object, we call another method defined in the `Iterator` trait — method [map](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.map):

```rust
fn map<B, F>(self, f: F) -> Map<Self, F>
where Self: Sized, F: FnMut(Self::Item) -> B,
{
    Map::new(self, f)
}
```
This method is very similar to filter, except it returns a [Map](https://doc.rust-lang.org/std/iter/struct.Map.html) object:

```rust
pub struct Map<I, F> {
    pub(crate) iter: I,
    f: F,
}
```

`Map` is also a wrapper around an iterator, holding the function used to transform the elements of the wrapped iterator.

4\) Finally, we call the [collect()](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.collect) method on the `Map` object. <ins>Only at this moment</ins> does the chain of wrappers begin to unwind and calculate the result. The result type depends on the generic type argument passed to `collect()`.

This "Matryoshka doll" of a vector-iterator wrapped in a `Filter`, which is then wrapped in a `Map`, looks something like this:

![](img/iterator_nesting.svg)

n the example above, we processed a vector and received the result as a vector (because we passed `Vec` as the type argument to `collect`). If we wanted the resulting elements in a [HashSet], we would simply specify that collection type in the `collect()` method.

```rust
use std::collections::HashSet;

fn main() {
    let v1: Vec<i32> = vec![1,2,3,4,5];
    let v2: HashSet<i32> = v1.into_iter()
        .filter(|x| x % 2 == 0)
        .map(|x| x * x)
        .collect::<HashSet<_>>();
    println!("{v2:?}"); // {16, 4}
}
```

Most collections in the standard library can produce an iterator, and most can be constructed from an iterator using `collect`. This makes iterators a universal API for processing and transforming collection types.

![](img/iterator_for_collections.svg)

Most third-party library collections also work seamlessly with iterators.

## Folds

An iterator doesn't have to be used to create another collection; it also allows you to "fold" (aggregate) elements. For this, the `Iterator` trait provides the [fold](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.fold) method:

```rust,noplayground
fn fold<B, F>(mut self, init: B, mut f: F) -> B
    where F: FnMut(B, Self::Item) -> B
{ ... }
```

This method takes two arguments:

1. An initial value to start the aggregation.
2. An accumulating function that takes the current accumulated value and the next element from the iterator, returning the updated result.

For example, let's sum the elements of an array using a fold:

```rust
let arr = [1, 2, 3];
let sum = arr.into_iter()
    .fold(0, |x, y| x + y);
println!("{sum}"); // 6
```

The folding process looks roughly like this:

![](img/iterator_fold.svg)

You might wonder: why do we need an initial value of `0` if we could just use the first element as the starting point?

First, the `Iterator` trait actually includes a [reduce](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.reduce) method. It works almost exactly like `fold`, but it uses the first element of the collection as the initial value.

```rust,edition2024
# fn main() {
let arr = [1, 2, 3];
let sum: Option<i32> = arr.into_iter()
    .reduce(|x, y| x + y);
println!("{sum:?}"); // Some(6)
# }
```

(`reduce` returns an `Option` because the collection might be empty)

Second, when using `fold`, the result of the aggregation can be a <ins>different type</ins> than the items being iterated. For instance, we can use `fold` to iterate over an array of strings and count the total number of characters:

```rust
let arr = ["aa", "bbb", "cccc"];
let char_count = arr.iter()
    .fold(0, |count, s| count + s.len());
println!("{char_count}"); // 9
```

***

It is also worth noting that the iterator has several [specializations](generics.md#specializaciya-treita).

For example, iterators over numeric types have an additional [sum](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.sum) method:

```rust
fn main() {
    let arr = [1, 2, 3];
    let sum: i32 = arr.into_iter().sum();
    println!("{sum}");
}
```

## Other Methods

### filter_map

The [filter_map](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.filter_map) method works like a standard `map`, but the transformation function it takes must return a value wrapped in an `Option`.

* If the function returns `Some(value)`, the value is automatically unwrapped and passed "further down the iterator pipeline."
* If the function returns `None`, that element is simply discarded.

Example: Getting the square roots of only those elements from which a root can be extracted.

```rust,edition2024
fn safe_sqrt(n: f32) -> Option<f32> {
    if n < 0.0 {
        None
    } else {
        Some(n.sqrt())
    }
}

fn main() {
    let arr = [4.0, -25.0, 9.0];
    let result = arr.into_iter()
        .filter_map(safe_sqrt)
        .collect::<Vec<_>>();
    println!("{result:?}"); // [2.0, 3.0]
}
```

### find

The [find](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.find) method takes a predicate and returns the first element that satisfies it.

Example: Finding the first even element.

```rust,edition2024
let arr = [1, 3, 5, 7, 8, 9];
let first_even: Option<i32> = arr.into_iter()
    .find(|x| x % 2 == 0);
println!("{first_even:?}");
```

The method returns an `Option` because there is a chance that no element satisfies the predicate.

> [!TIP]
> While the `Iterator` trait defines many useful methods, you may find you need even more in practice. The excellent [itertools](https://crates.io/crates/itertools) library provides a vast array of additional iterator methods.


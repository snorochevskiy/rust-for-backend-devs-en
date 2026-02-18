# Collections

The Rust standard library provides the following collection types:

* [Vec\<T>](https://doc.rust-lang.org/std/vec/struct.Vec.html) — Vector: we are already familiar with this one.
* [LinkedList\<T>](https://doc.rust-lang.org/std/collections/struct.LinkedList.html) — Doubly linked list.
* [VecDeque\<T>](https://doc.rust-lang.org/std/collections/struct.VecDeque.html) — A growable ring buffer. It behaves like a vector but allows efficient element addition to both the front and the back.
* [HashMap\<R,V>](https://doc.rust-lang.org/std/collections/struct.HashMap.html) — Hash map.
* [HashSet\<T>](https://doc.rust-lang.org/stable/std/collections/struct.HashSet.html) — Hash set.
* [BTreeMap\<K,V>](https://doc.rust-lang.org/stable/std/collections/struct.BTreeMap.html) — A key-value dictionary based on the B-Tree structure.
* [BinaryHeap\<T>](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html) — A priority queue based on a [binary heap](https://en.wikipedia.org/wiki/Binary_heap).

## Vec

We already covered the vector and its memory layout in the [Vector](../rust-basics/vector.md) chapter.

Let's take a look at examples of some frequently used methods.

```rust
use std::cmp::Ordering;

fn main() {
    // Create a vector with a buffer for 10 elements
    let mut v: Vec<i32> = Vec::with_capacity(10);

    // Add elements
    v.push(0);
    v.push(4);
    v.push(1);
    v.push(3);
    v.push(2);

    // Add multiple elements from a slice at once
    v.extend_from_slice(&[5, 6]);

    // Add multiple elements from an iterator at once
    v.extend([8, 7, 9].iter());

    println!("{v:?}"); // [0, 4, 1, 3, 2, 5, 6, 8, 7, 9]

    // Sort the vector (ascending)
    v.sort();

    println!("{v:?}"); // [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

    // Sort descending
    v.sort_by(|a, b| {
        if a > b {
            Ordering::Less
        } else if a < b {
            Ordering::Greater
        } else {
            Ordering::Equal
        }
    });

    println!("{v:?}"); // [9, 8, 7, 6, 5, 4, 3, 2, 1, 0]

    // Extract the last element from the vector
    println!("Last: {:?}", v.pop()); // Last: Some(0)

    v.extend_from_slice(&[1, 2, 3]);

    println!("{v:?}"); // [9, 8, 7, 6, 5, 4, 3, 2, 1, 1, 2, 3]

    // Remove consecutive duplicates
    v.dedup();

    println!("{v:?}"); // [9, 8, 7, 6, 5, 4, 3, 2, 1, 2, 3]
}

```

## LinkedList

`LinkedList` is a classic doubly linked list.

Each element is represented by a `Node` type — a struct with three fields:

* The element's value.
* A pointer to the previous element in the list.
* A pointer to the next element.

The `LinkedList` itself is a struct with these fields:

* A pointer to the first element.
* A pointer to the last element.
* The length of the list.

![](img/collections-linked_list.svg)

Unlike a vector, a doubly linked list allows for efficient insertion or deletion in the middle or at the beginning of the list.

Example:

```rust
use std::collections::LinkedList;

fn main() {
    let mut list = LinkedList::<i32>::new();
    list.push_back(1); // [1]
    list.push_back(2); // [1,2]
    list.push_back(3); // [1,2,3]
    list.push_front(0); // [0,1,2,3];

    println!("{list:?}"); // [0, 1, 2, 3]

    let last: Option<i32> = list.pop_back();
    println!("{:?}", last); // Ok(3)

    let first: Option<i32> = list.pop_front();
    println!("{:?}", first); // 0

    println!("{list:?}"); // [1, 2]
}
```

## VecDeque

A doubly linked list allows for element insertion and deletion in the middle and at the beginning, but its disadvantage is that accessing an element by index has a complexity of O(n/2), unlike the O(1) complexity of a vector. The `VecDeque` collection aims to address this.

`VecDeque` is implemented as a <ins>ring buffer</ins>, which allows for efficient addition to the front and relatively efficient addition to the middle. At the same time, `VecDeque`, like a vector, maintains O(1) complexity for indexed access.

![](img/collections-vecdeque.svg)

Usage example:

```rust
use std::collections::VecDeque;

fn main() {
    let mut v = VecDeque::from([1, 2, 3]);
    v.insert(2, 99);
    v.push_front(0);
    println!("{v:?}"); // [0, 1, 2, 99, 3]

    let last: Option<i32> = v.pop_back();
    println!("{:?}", last); // Ok(3)

    let first: Option<i32> = v.pop_front();
    println!("{:?}", first); // 0

    println!("{v:?}"); // [1, 2, 99]
}
```

## HashMap

`HashMap<K,V>` is a dictionary that stores key-value pairs as a hash table.

Elements are stored in a contiguous buffer divided into sections called **buckets**.

When inserting a new pair:

1. A hash code is calculated for the key.
2. A `bucket_mask` is applied to the hash code to determine which bucket will store the pair.
3. The specific position in the buffer is determined using the bucket number (and the "Control Bytes" preamble at the beginning of the buffer).
4. If the number of elements in the bucket exceeds a certain limit (`growth_left`) after insertion, a new, larger buffer is allocated, and the values are moved over.

![](img/collections-hash_amp.svg)

The key type must implement the `Eq` and `Hash` traits, while the value type must implement `PartialEq`.

Usage example:

```rust
use std::collections::HashMap;

fn main() {
    let mut h: HashMap<i32, String> = HashMap::new();
    h.insert(1, "one".to_string());
    h.insert(2, "two".to_string());
    h.insert(3, "three".to_string());
    h.insert(4, "three".to_string());
    h.insert(5, "three".to_string());

    println!("Pairs:");
    for (key, value) in h.iter() {
        println!("  {key} -> {value}");
    }

    println!("Keys:");
    for key in h.keys() {
        println!("  {key}");
    }

    println!("Values:");
    for value in h.values() {
        println!("  {value}");
    }

    println!("Getting reference to value for key=1:");
    if let Some(reference) = h.get(&1) {
        println!("  {reference}");
    }

    println!("Extracting value for key=1:");
    if let Some(value) = h.remove(&1) {
        println!("  {value}");
    }

    println!("Contains key 1: {}", h.contains_key(&1));
}
```

## HashSet

`HashSet<T>` is a hash set that allows for fast element insertion and searching. It is implemented as a wrapper around `HashMap<T, ()>`, so everything applicable to `HashMap` applies to `HashSet` as well.

Usage example:

```rust
use std::collections::HashSet;

fn main() {
    let mut s: HashSet<String> = HashSet::new();
    s.insert("one".to_string());
    s.insert("two".to_string());
    s.insert("three".to_string());

    for value in s.iter() {
        println!("{value}");
    }

    if let Some(reference) = s.get(&"one".to_string()) {
        println!("{reference}");
    }

    if let Some(value) = s.take(&"one".to_string()) {
        println!("{value}");
    }

    println!("HashSet contains 'one': {}", s.contains(&"one".to_string()));
}
```

## BTreeMap

`BTreeMap<K,V>`, like `HashMap<K,V>`, is a dictionary that stores key-value pairs. The main difference is that `BTreeMap` stores data in a **B-Tree** structure.

![](img/collections-btree_map.svg)

From an API perspective, `BTreeMap` is almost identical to `HashMap`: in the `HashMap` example, you could replace the dictionary type with `BTreeMap`, and the program would still compile.

The only technical difference: for `HashMap`, the key type must implement `Hash` and `Eq`, whereas for `BTreeMap`, the key type must implement the `Ord` trait.

Insertion and search operations for `BTreeMap` are slightly slower than for `HashMap`, especially with large dictionaries (over a thousand elements). However, `BTreeMap` keeps elements sorted by key, which can be useful in many scenarios.

## BinaryHeap

`BinaryHeap<T>` is a priority queue implemented as a binary heap.

The heap is built on top of a vector.

![](img/collections-binary_heap.svg)

> [!NOTE]
> If you aren't familiar with a binary heap, it might look like just another binary tree. However, it isn't. In a binary search tree, the values in the left branch must be smaller than the values in the right branch. A heap has no such restriction. The only rule is that the value in a parent node must be greater than (or smaller than, depending on the heap type) the values in its children.

When adding new elements or removing existing ones, the heap rearranges itself so that the top node always contains the largest element. By repeatedly popping the top element, we get a sequence sorted in descending order.

Example of usage:

```rust
use std::collections::BinaryHeap;

fn main() {
    let mut queue: BinaryHeap<i32> = BinaryHeap::new();
    queue.push(2);
    queue.push(7);
    queue.push(1);
    queue.push(4);
    queue.push(9);

    while let Some(element) = queue.pop() {
        print!(" {element}"); // 9 7 4 2 1
    }
}
```

Since `BinaryHeap` orders elements by comparison, the element type must implement the `Ord` trait.

# Bestie Standard Library — Algorithms (`bestie.lib.algorithms`)

This document defines the **general-purpose algorithms** provided by Bestie’s standard library.

Algorithms in Bestie are:

* **Explicit**
* **Generic**
* **Allocation-aware**
* **Deterministic**
* **Zero-surprise**

They operate on **collections, iterators, and ranges**, without introducing hidden memory allocation, implicit copying, or runtime polymorphism.

---

## 1. Design Philosophy

Bestie algorithms follow strict rules:

1. **Algorithms do not own data**
2. **No implicit allocation**
3. **No hidden mutation**
4. **Order, stability, and cost are explicit**
5. **Dispatch is compile-time**

Algorithms are **functions**, not methods.
They are reusable, composable, and predictable.

---

## 2. Supported Algorithm Categories

`bestie.lib.algorithms` provides:

* Ordering & searching
* Aggregation
* Partitioning
* Transformation
* Iteration helpers

These algorithms operate on:

* `Iterable<T>`
* `Iterator<T>`
* Std-lib collections (`set`, `map`, `deque`, `heap`)
* Core `list<T>`

---

## 3. Sorting Algorithms

### 3.1 `sort`

```bestie
fun sort<T impl Comparable>(data: var list<T>)
```

Purpose:

* Sorts elements **in-place**
* Order is **not stable**

Properties:

* Mutates input
* No allocation
* Requires `Comparable`

Example:

```bestie
var nums = list<int>.of(4, 1, 3)
sort(nums)
```

Use when:

* Performance matters more than stability

---

### 3.2 `stableSort`

```bestie
fun stableSort<T impl Comparable>(data: var list<T>)
```

Purpose:

* Stable in-place sorting
* Preserves relative order of equal elements

Properties:

* May allocate temporary buffers
* Stability is guaranteed
* Deterministic ordering

Use when:

* Order preservation matters

---

## 4. Searching Algorithms

### 4.1 `binarySearch`

```bestie
fun binarySearch<T impl Comparable>(
    data: list<T>,
    target: T
): option<int>
```

Rules:

* Input **must be sorted**
* Returns index wrapped in `option`

Example:

```bestie
val idx = binarySearch(nums, 3)

switch (idx) {
    case option.Present(val i) => print(i)
    case option.Not_Present    => print("not found")
}
```

Failure is represented by `Not_Present`, not `-1`.

---

## 5. Min / Max Utilities

### 5.1 `min`

```bestie
fun min<T impl Comparable>(a: T, b: T): T
```

### 5.2 `max`

```bestie
fun max<T impl Comparable>(a: T, b: T): T
```

Rules:

* Pure functions
* No allocation
* Compile-time dispatch

---

### 5.3 `clamp`

```bestie
fun clamp<T impl Comparable>(
    value: T,
    lower: T,
    upper: T
): T
```

Guarantees:

* Result ∈ [lower, upper]
* Deterministic comparison

---

## 6. Partitioning

### 6.1 `partition`

```bestie
fun partition<T>(
    data: var list<T>,
    predicate: fn(T) -> bool
): int
```

Purpose:

* Reorders elements so that:

  * Predicate-true elements come first
* Returns partition index

Properties:

* In-place
* Unstable
* No allocation

Example:

```bestie
val idx = partition(nums, x => x % 2 == 0)
```

---

## 7. Folding & Reduction

### 7.1 `fold`

```bestie
fun fold<T, R>(
    data: Iterable<T>,
    initial: R,
    op: fn(R, T) -> R
): R
```

Purpose:

* General reduction
* Left-associative

Example:

```bestie
val sum = fold(nums, 0, (acc, x) => acc + x)
```

Rules:

* No mutation unless `R` is mutable
* No implicit allocation

---

## 8. Zipping

### 8.1 `zip`

```bestie
fun zip<A, B>(
    a: Iterable<A>,
    b: Iterable<B>
): Iterator<(A, B)>
```

Rules:

* Stops at shortest iterable
* Lazy evaluation where possible
* No copying unless consumed

Example:

```bestie
for (x, y) in zip(xs, ys) {
    print(x, y)
}
```

---

## 9. Error & Safety Guarantees

Algorithms in `bestie.lib.algorithms`:

* Never use exception-style control flow
* Never hide allocation
* Never assume mutability
* Never cross thread boundaries implicitly

Misuse (e.g. binarySearch on unsorted data) is a **logic error**, not undefined behavior.

---

## 10. Relationship with Functional Utilities

`bestie.lib.algorithms` complements:

* `bestie.lib.functional` (`map`, `filter`, etc.)
* `patterns.Iterator`
* `patterns.Iterable`

Algorithms are **foundational**, not syntactic sugar.

---

## 11. What Is Deliberately Excluded

Not included in `bestie.lib.algorithms`:

* Parallel algorithms
* Lazy infinite streams
* Implicit SIMD
* Auto-vectorization hints
* Runtime specialization

These belong to:

* `ext.concurrency`
* `ext.simd`
* Domain-specific libraries

---

## 12. Summary

`bestie.lib.algorithms` is:

* Predictable
* Fast
* Explicit
* Allocation-aware
* Free of magic

It gives Bestie:

* The power of STL-style algorithms
* Without the ambiguity
* Without runtime surprises

Algorithms do one thing.
They do it well.
And they tell you exactly what they cost.

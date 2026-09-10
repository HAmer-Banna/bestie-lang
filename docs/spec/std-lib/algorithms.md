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

### How these signatures are written

Collection variations are invariant (`collections.md` §3.3), so a parameter naming `list<T>` accepts array-backed lists and nothing else. Every algorithm here therefore falls into one of two shapes:

* **Read-only traversal** takes a generic parameter constrained by `Iterable<T>` — `fun <C impl Iterable<T>> f(xs: C)`. This accepts every collection variation, plus core `array<T>`, `slice<T>`, and `range<T>`. It is fully monomorphized, so each instantiation compiles to what a hand-written loop would.
* **In-place reordering and indexed access** names a concrete type, because the cost is part of the contract. `binarySearch` says `list<T>` precisely because O(log n) depends on array-backed indexing; the same signature over `list<T>.linked` would be O(n log n) with nothing in the source saying so.

In-place algorithms take `ptr<list<T>>`, not `list<T>`: `list<T>` is a `class` and classes are not copyable across a call (`core/memory.md` §6.1). Pointing is how a callee mutates the caller's object.

---

## 3. Sorting Algorithms

### 3.1 `sort`

```bestie
fun <T impl Comparable> sort(data: ptr<list<T>>)
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
var nums = list<int>.new(4, 1, 3)
sort(nums)
```

Use when:

* Performance matters more than stability

---

### 3.2 `stableSort`

```bestie
fun <T impl Comparable> stableSort(data: ptr<list<T>>)
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
fun <T impl Comparable> binarySearch(
    data: list<T>,
    target: T
): int ?
```

Rules:

* Input **must be sorted**
* Returns `int ?` — the index, or absent

Example:

```bestie
if (val i = binarySearch(nums, 3)) {
    print(i)
} else {
    print("not found")
}
```

A miss is represented by absence, not by `-1`.

---

## 5. Min / Max Utilities

### 5.1 `min`

```bestie
fun <T impl Comparable> min(a: T, b: T): T
```

### 5.2 `max`

```bestie
fun <T impl Comparable> max(a: T, b: T): T
```

Rules:

* Pure functions
* No allocation
* Compile-time dispatch

---

### 5.3 `clamp`

```bestie
fun <T impl Comparable> clamp(
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
fun <T> partition(
    data: ptr<list<T>>,
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
fun <C impl Iterable<T>, T, R> fold(
    data: C,
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
fun <CA impl Iterable<A>, CB impl Iterable<B>, A, B> zip(
    a: CA,
    b: CB
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

* `bestie.lib.concurrency`
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

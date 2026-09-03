# Bestie Math Module

This document defines the **core mathematical abstractions** provided by Bestie and the separation between **core numeric types** and **standard-library mathematical utilities**.

Math in Bestie is:

* Deterministic
* Runtime-independent
* Explicit
* Compile-time friendly

Bestie deliberately avoids implicit numeric magic or operator-heavy abstractions that obscure cost or behavior.

---

## 1. Scope and Layering

Bestie math is divided into two layers:

1. **Core Math (language level)**
2. **Math Std-Lib (`bestie.lib.math`)**

Only primitives that must be understood by the compiler live in core. Everything else is library code.

---

## 2. Core Numeric Types

Core numeric primitives match the **Primitive Types** section of the [core language spec](../core/lang.md). In short:

* **Signed integers:** `int8`, `int16`, `int32`, `int64`, and pointer-sized `int`
* **Unsigned integers:** `uint8`, `uint16`, `uint32`, `uint64`, and pointer-sized `uint` (`byte` is an alias for `uint8`)
* **Floating-point:** `float32` and `float64` (IEEE 754)

### 2.1 Design Rules

* All numeric types have **well-defined size and overflow behavior**
* No implicit widening or narrowing
* Conversions are **explicit**
* No boxed numbers or hidden heap allocation

```bestie
val x : int = 10
val y : int64 = x as int64   // explicit
```

---

## 3. Arithmetic Semantics

* Arithmetic is **strict and predictable**
* Signed integer overflow is **defined** (wrap in release, trap in debug; see core spec), not undefined behavior
* Overflow on constant expressions may be caught at compile time where provable

```bestie
val x : int = 2_000_000_000
val y = x + x   // compile-time error or explicit overflow handling
```

---

## 4. `matrix<T>`

Core has no matrix type. Compile-time grids that are **storage only** stay as nested `array<T>` (`array<int>[3][3]` in `core/types.md`). Algebra is not a language primitive.

`matrix<T>` lives in `bestie.lib.math`. It is a **`class`**, lowercase like `list` and `option`: a foundational library type, not a nominal domain object.

### 4.1 Class kind

| Type | Kind | Reasoning |
| ---- | ---- | --------- |
| `matrix<T>` | `class` | Owns a heap buffer whose size is `rows * cols` at runtime; has identity; supports in-place mutation |

It is not a `value class`: those cannot hold `own` fields, are copy-by-value, and cannot own a runtime-sized buffer. Copying a matrix on assignment would silently duplicate storage and hide allocation.

It is not a collection. Dimensions are fixed at construction. There is no `.add`, no builder growth, and no `list<T>.matrix` view.

`T` is a core numeric primitive (`int*`, `uint*`, `float32`, `float64`).

### 4.2 Layout and construction

Storage is a single contiguous **row-major** buffer. Index `(r, c)` is `r * cols + c`. No pointer-to-pointer rows.

```bestie
import bestie.lib.math

val a = matrix<float64>.zeros(3, 3)
val b = matrix<float64>.identity(3)
val c = matrix<float64>.of(2, 2, {1.0, 2.0, 3.0, 4.0})  // row-major, length == rows * cols
```

Rules:

* `rows` and `cols` must be `> 0` — a zero dimension is a panic (violated invariant)
* `.of` panics if the literal length is not `rows * cols`
* Out-of-range `get` / `set` is a panic, same as `array<T>`

### 4.3 Operations

Higher-level math uses **named methods**, not `*` / `+` overloads (see §5).

```bestie
import bestie.lib.math

val a = matrix<float64>.identity(3)
val b = matrix<float64>.zeros(3, 3)
b.set(0, 1, 2.0)

val c = a.mul(b)          // new matrix; allocation is at this call
val d = a.add(b)
val t = a.transpose()

a.mulInPlace(b)           // mutates `a`; no extra allocation
```

Shape mismatches on `mul` / `add` / `mulInPlace` are `! DimensionError` — the sizes are data, not a broken invariant of a single matrix.

```bestie
errors DimensionError {
    IncompatibleShape
}
```

Indexing:

```bestie
val x = a.get(0, 1)
a.set(0, 1, 3.5)
val rows = a.rows()
val cols = a.cols()
```

### 4.4 What `matrix<T>` is not

* Not a replacement for `array<T>[n][m]` — use arrays when the size is compile-time and you only need storage
* Not a tensor library — higher rank stays out of this module until a concrete need exists
* Not a collection — it does not implement the collections builder API

SIMD or hardware acceleration may apply inside the implementation; it is not exposed as a second type.

---

## 5. Operators vs Functions

Core math favors **explicit functions** over overloaded operators.

Operators are limited to:

* Basic arithmetic (`+`, `-`, `*`, `/`)
* Comparison

Higher-level math uses named functions:

```bestie
import bestie.lib.math

val x = sqrt(16)
val y = pow(2, 8)
```

---

## 6. Determinism and Performance

Math operations in Bestie:

* Have no hidden allocations
* Are deterministic across platforms
* Can be evaluated at compile time when inputs are constant

```bestie
const x: int = pow(2, 10)  // resolved at compile time
```

---

## 7. What Math Does Not Include

Core and std-lib math intentionally exclude:

* Symbolic math
* Implicit vectorization
* Runtime-dependent optimizations
* Lazy evaluation

These may be built as external libraries.

---

## 8. Summary

Bestie math design:

* Keeps core minimal and predictable
* Gives algebra its own type (`matrix<T>`), separate from `array<T>` storage
* Pushes advanced math into explicit libraries
* Favors clarity over cleverness

> Math in Bestie is **explicit by default and powerful by choice**.

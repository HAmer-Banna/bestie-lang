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

## 4. No Core Matrix Type

Bestie **does not define a matrix type in core**.

This is intentional.

Reasons:

* Matrix semantics vary widely (layout, mutability, dimensionality)
* Std-lib collections already provide structured storage
* Mathematical matrices are domain-specific, not language primitives

As a result, **std-lib collections such as `list<T>.matrix` use a C-style flat layout** only as a storage optimization, not as a mathematical abstraction.

```bestie
val m : list<int>.matrix = list<int>.matrix.of(3, 3)
```

This representation:

* Is row-major
* Uses contiguous memory
* Has no algebraic meaning by itself

---

## 5. Linear Algebra Support (Std-Lib)

True matrix and linear algebra support lives in:

```
bestie.lib.math
```

### 5.1 Linear Algebra Structures

The std-lib introduces **dedicated types** for mathematical matrices, intentionally named to avoid confusion with collections.

Examples (illustrative):

* `tensor2d<T>`
* `tensor3d<T>`
* `linalg2<T>`

These types:

* Encode dimensionality in the type
* Support algebraic operations
* May leverage SIMD or hardware acceleration
* Are independent of collection APIs

```bestie
import bestie.lib.math

val a = tensor2d<int>(3, 3)
val b = tensor2d<int>(3, 3)
val c = a.mul(b)
```

---

## 6. Operators vs Functions

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

## 7. Determinism and Performance

Math operations in Bestie:

* Have no hidden allocations
* Are deterministic across platforms
* Can be evaluated at compile time when inputs are constant

```bestie
const val x = pow(2, 10)  // resolved at compile time
```

---

## 8. What Math Does Not Include

Core and std-lib math intentionally exclude:

* Symbolic math
* Implicit vectorization
* Runtime-dependent optimizations
* Lazy evaluation

These may be built as external libraries.

---

## 9. Summary

Bestie math design:

* Keeps core minimal and predictable
* Avoids confusing collection storage with algebra
* Pushes advanced math into explicit libraries
* Favors clarity over cleverness

> Math in Bestie is **explicit by default and powerful by choice**.

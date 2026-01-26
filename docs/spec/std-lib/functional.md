# Bestie Standard Library — Functional Utilities (functional.md)

This document defines the **functional utilities** provided by the Bestie standard library.

Functional utilities in Bestie are:

* Explicit
* Compile-time resolvable
* Allocation-aware
* Ownership-safe
* Free of hidden runtime behavior

They are designed to **compose with core collections**, not replace them, and to preserve Bestie’s core rhythm: *what happens in code is exactly what happens in memory*.

---

## 1. Design Philosophy

Bestie functional utilities deliberately avoid:

* Lazy magic
* Implicit allocations
* Hidden iterators
* Runtime closures with captured heap state

Instead, they provide:

* Explicit transformation pipelines
* Compiler-visible ownership flow
* Predictable allocation behavior
* Zero-cost abstractions where possible

Golden Rule:

> If a functional operation allocates, moves, or copies data, it must be visible in the type signature.

---

## 2. Scope of Functional Utilities

Functional utilities are provided as **free functions**, not methods.

Reasons:

* Avoid bloating core collection APIs
* Preserve static dispatch
* Enable specialization by the compiler
* Prevent implicit `this` capture

All functions live under:

```
std::functional
```

---

## 3. Core Functional Operations

### 3.1 map

Transforms each element of a collection into a new value.

Signature:

```
fun map<T, U>(
    src: ref List<T>,
    f: fn(T) -> U
): List<U>
```

Rules:

* Always produces a new collection
* Allocation is explicit and visible
* Source collection is never mutated
* Function `f` must be pure (no side effects enforced by convention)

Example:

```
val nums = List.of(1, 2, 3)
val squares = map(nums, fn(x: int) -> int { x * x })
```

---

### 3.2 filter

Selects elements based on a predicate.

Signature:

```
fun filter<T>(
    src: ref List<T>,
    predicate: fn(T) -> bool
): List<T>
```

Rules:

* Produces a new collection
* Preserves element order
* Does not mutate the source
* Allocation behavior is explicit

Example:

```
val evens = filter(nums, fn(x: int) -> bool { x % 2 == 0 })
```

---

### 3.3 fold (reduce)

Aggregates elements into a single value.

Signature:

```
fun fold<T, R>(
    src: ref List<T>,
    init: R,
    f: fn(R, T) -> R
): R
```

Rules:

* No allocation by default
* Deterministic traversal order
* Suitable for numeric and structural reduction

Example:

```
val sum = fold(nums, 0, fn(acc: int, x: int) -> int { acc + x })
```

---

### 3.4 forEach

Applies a function to each element for side effects.

Signature:

```
fun forEach<T>(
    src: ref List<T>,
    f: fn(T) -> void
): void
```

Rules:

* Does not allocate
* Used for IO or mutation outside the collection
* Explicitly signals side effects

Example:

```
forEach(nums, fn(x: int) { print(x) })
```

---

## 4. Allocation-Aware Variants

### 4.1 mapInto (Arena-backed)

Maps elements into a destination arena.

Signature:

```
fun mapInto<T, U>(
    src: ref List<T>,
    arena: ref Arena,
    f: fn(T) -> U
): List<U>
```

Rules:

* All allocations occur inside the provided arena
* Returned list is invalid after arena.reset() or arena.release()
* Enables batch allocation patterns

Example:

```
val arena = Arena.of(1, MB)
val result = mapInto(nums, arena, fn(x: int) -> int { x * 2 })
```

---

## 5. Ownership & Lifetime Rules

* Functional utilities never steal ownership implicitly
* Returned collections own their elements unless arena-backed
* `ref` inputs guarantee no mutation or ownership transfer
* Arena-backed results are explicitly scoped

Invalid cases (compile-time errors):

* Returning arena-backed collections from longer-lived scopes
* Capturing owned values inside functional callbacks

---

## 6. No Lazy Evaluation

Bestie does **not** provide lazy streams in core std-lib.

Reasons:

* Lazy evaluation hides control flow
* Breaks predictability of allocation
* Complicates debugging and reasoning

Lazy abstractions may exist in **opt-in libraries**, never in core std-lib.

---

## 7. Error Handling Integration

Functional utilities compose with:

* `Option<T>`
* `Result<T, E>`

Example:

```
val valid = filter(users, fn(u: User) -> bool {
    u.age > 18
})
```

No implicit short-circuiting is performed.

Explicit variants may exist:

* `mapResult`
* `filterPresent`

These must be named explicitly.

---

## 8. What Bestie Deliberately Avoids

* Implicit iterators
* Monad syntax sugar
* Operator overloading for pipelines
* Runtime closures with captured heap state
* Hidden allocation via lambdas

---

## 9. Summary

Bestie functional utilities:

* Favor explicitness over cleverness
* Preserve ownership and lifetime guarantees
* Compose naturally with core collections
* Scale from backend logic to systems code

They provide the *benefits* of functional programming without sacrificing the **predictability required for system-level correctness**.

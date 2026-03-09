# Bestie Standard Library — Utility Package

This document defines the **utility package** of the Bestie standard library. These types form the foundation for memory management, error modeling, and structural interoperability. All utilities are explicit, predictable, and compiler-verifiable.

This package is part of the **core**. It does not depend on threading primitives, atomics, or runtime services beyond threadOs / threadLight.

---

## 1. StringBuilder

`StringBuilder` is the canonical utility for efficient string construction.

### Design

* Mutable
* Identity-based
* Explicit allocation behavior
* No implicit copies

`StringBuilder` is intended for performance-sensitive paths where repeated string concatenation would otherwise cause unnecessary allocations.

### Characteristics

* Backed by a contiguous buffer
* Growth strategy is deterministic
* Conversion to `str` is explicit

```bestie
var sb = StringBuilder.new()
sb.append("Hello")
sb.append(" ")
sb.append("World")

val s = sb.toString()
```

---

## 2. Arena

`Arena` is a **value class** that provides region-based allocation.

It allows fast allocation and bulk deallocation with well-defined lifetime semantics.

### Construction

```bestie
val arena = Arena.of(1, MB)
```

* `size: int` — numeric size
* `unit: SizeUnit` — enum (`KB`, `MB`, `GB`)

### Allocation

```bestie
arena.add(42)
arena.add(listOfInts)
```

* `add(T)` allocates a single value
* `add(list<T>)` allocates a contiguous sequence

All allocations belong to the arena and share the same lifetime.

### Lifetime Control

```bestie
arena.reset()   // reuse memory
arena.release() // invalidate arena
```

* `reset()` clears all allocations but keeps the arena usable
* `release()` permanently invalidates the arena; further use is a compile-time error

### Rules

* Arenas do not move memory
* Arenas do not escape their owning scope
* Arenas do not participate in ownership transfer

---

## 3. Option<T>

`Option<T>` represents **explicit presence or absence** of a value.

Bestie does not use null, none, or nil.

### Definition

```bestie
enum class Option<T> {
  Present(T)
  Not_Present
}
```

### Usage

```bestie
fun findUser(id: int): Option<User> {
  if exists(id) {
    return Option.Present(loadUser(id))
  }
  return Option.Not_Present
}
```

### Properties

* Fully type-safe
* Exhaustive pattern matching enforced
* No implicit unwrapping

---

## 4. Result<T, E>

`Result<T, E>` represents an operation that may succeed or fail with a typed error.

### Definition

```bestie
enum class Result<T, E> {
  Ok(T)
  Err(E)
}
```

### Usage

```bestie
fun parseInt(s: str): Result<int, ParseError> {
  if valid(s) {
    return Result.Ok(convert(s))
  }
  return Result.Err(ParseError.InvalidFormat)
}
```

### Guidelines

* Prefer `Result` for expected, recoverable failures
* Do not mix `Result` and `throw` without justification

---

## 5. Equable Protocol

`Equable` defines **structural equality** between two values of the same type.

It is the foundational protocol behind `==` semantics in Bestie.

### Definition

```bestie
protocol Equable<T> {
  fun equal(other: T): bool
}
```

### Semantics

* `equal` must be **reflexive**, **symmetric**, and **transitive`
* Equality is **value-based**, not identity-based
* The method must not mutate either operand
* Resolution is static and compile-time driven

### Interaction with `==`

If a type uses `impl Equable`, the `==` operator is lowered to:

```bestie
lhs.equal(rhs)
```

If `Equable` is not implemented:

* Value types fall back to compiler-generated structural comparison
* Reference types fall back to identity comparison

No dynamic dispatch is introduced.

---

## 6. Comparable Protocol

`Comparable` defines a total ordering between values.

### Definition

```bestie
protocol Comparable<T> {
  fun compareTo(other: T): int
}
```

### Rules

* Must be antisymmetric, transitive, and total
* Used by sorting and ordering utilities
* Resolved at compile time

---

## 7. Hashable Protocol

`Hashable` defines a stable hash for a value and uses **`ext Equable`**.

### Definition

```bestie
protocol Hashable ext Equable<T> {
  fun hash(): int
}
```

### Rules

* If `a == b` is true, then `a.hash() == b.hash()` must be true
* Hash must be stable for the value’s lifetime
* `equal` defines equality semantics; `hash` must be consistent with it
* No runtime reflection

`Hashable` is used by hash-based collections and lookup structures.

---

## Summary

The utility package provides:

* Canonical utility for efficient string construction (`StringBuilder`)
* Explicit memory construction (`Arena`)
* Explicit absence modeling (`Option`)
* Typed failure (`Result`)
* Structural equality (`Equable`)
* Ordering contracts (`Comparable`)
* Hash-based identity (`Hashable`)

These utilities establish the rhythm of Bestie’s standard library: **explicit, orthogonal, and compiler-verifiable**.

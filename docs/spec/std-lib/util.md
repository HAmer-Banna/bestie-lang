# Bestie Standard Library — Utility Package

This document defines the **utility package** of the Bestie standard library. These types form the foundation for error modeling and structural interoperability. All utilities are explicit, predictable, and compiler-verifiable.

Bestie uses lowercase for foundational abstractions such as `option<T>` and `result<T,E>`, while nominal concrete utility types such as `StringBuilder` remain PascalCase.

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

## 2. option<T>

`option<T>` represents **explicit presence or absence** of a value.

Bestie does not use null, none, or nil.

### Definition

```bestie
enum option<T> {
  Present(T)
  Not_Present
}
```

### Usage

```bestie
fun findUser(id: int): option<User> {
  if exists(id) {
    return option.Present(loadUser(id))
  }
  return option.Not_Present
}
```

### Properties

* Fully type-safe
* Exhaustive pattern matching enforced
* No implicit unwrapping

---

## 3. result<T, E>

`result<T, E>` represents an operation that may succeed or fail with a typed error.

### Definition

```bestie
enum result<T, E> {
  Ok(T)
  Err(E)
}
```

### Usage

```bestie
fun parseInt(s: str): result<int, ParseError> {
  if valid(s) {
    return result.Ok(convert(s))
  }
  return result.Err(ParseError.InvalidFormat)
}
```

### Guidelines

* Prefer `result` for expected, recoverable failures
* Do not mix `result` and `throw` without justification

---

## 4. Equable Protocol

`Equable` defines **structural equality** between two values of the same type.

It is the foundational protocol behind `==` semantics in Bestie.

### Definition

```bestie
protocol Equable<T> {
  fun equal(other: T): bool
}
```

### Semantics

* `equal` must be **reflexive**, **symmetric**, and **transitive**
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

## 5. Comparable Protocol

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

## 6. Hashable Protocol

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
* Explicit absence modeling (`option`)
* Typed failure (`result`)
* Structural equality (`Equable`)
* Ordering contracts (`Comparable`)
* Hash-based identity (`Hashable`)

These utilities establish the rhythm of Bestie’s standard library: **explicit, orthogonal, and compiler-verifiable**.

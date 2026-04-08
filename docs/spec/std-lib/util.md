# Bestie Standard Library — Utility Package

This document defines the **utility package** of the Bestie standard library. These types form the foundation for error modeling and structural interoperability. All utilities are explicit, predictable, and compiler-verifiable.

Bestie uses lowercase for foundational abstractions such as `option<T>` and `result<T,E>`, while nominal concrete utility types such as `StringBuilder` remain PascalCase.

---

## 1. StringBuilder

`StringBuilder` is a **`class`** — the canonical utility for efficient string construction.

It is a `class` (not `value class` or `data class`) because:

* It has **mutable internal state** (a growable byte buffer and a write cursor)
* It has **identity** — two `StringBuilder` instances that produce the same string are still distinct objects
* It **owns its backing buffer**, which is heap-allocated and freed when the builder is freed

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
* Do not mix `result` and panic-based invariant handling without justification

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
protocol Hashable<T> ext Equable<T> {
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

---

## 7. Operator Overloading Protocols

Bestie supports operator overloading through **explicit protocols**. The compiler lowers operator expressions to protocol method calls at compile time — fully monomorphized, no runtime dispatch, no vtables.

All operator protocols are resolved at compile time. Using an operator on a type that does not implement the corresponding protocol is a **compile-time error**.

---

### 7.1 Arithmetic Operators

```bestie
protocol Addable<T> {
    fun add(other: T): T        // a + b
    fun addAssign(other: T)     // a += b
}

protocol Subtractable<T> {
    fun sub(other: T): T        // a - b
    fun subAssign(other: T)     // a -= b
}

protocol Multipliable<T> {
    fun mul(other: T): T        // a * b
    fun mulAssign(other: T)     // a *= b
}

protocol Divisible<T> {
    fun div(other: T): T        // a / b
    fun divAssign(other: T)     // a /= b
}

protocol Modulable<T> {
    fun mod(other: T): T        // a % b
    fun modAssign(other: T)     // a %= b
}

protocol Negatable {
    fun neg(): Self             // -a (unary)
}
```

---

### 7.2 Index Operators

```bestie
protocol Indexable<I, T> {
    fun get(index: I): T        // a[i]
}

protocol IndexAssignable<I, T> {
    fun set(index: I, val: T)   // a[i] = v
}
```

---

### 7.3 Lowering Rules

The compiler lowers operator syntax to protocol calls at compile time:

| Syntax   | Lowered to          |
| -------- | ------------------- |
| `a + b`  | `a.add(b)`          |
| `a - b`  | `a.sub(b)`          |
| `a * b`  | `a.mul(b)`          |
| `a / b`  | `a.div(b)`          |
| `a % b`  | `a.mod(b)`          |
| `-a`     | `a.neg()`           |
| `a += b` | `a.addAssign(b)`    |
| `a[i]`   | `a.get(i)`          |
| `a[i]=v` | `a.set(i, v)`       |

`==` and `!=` are lowered via `Equable`. `<`, `>`, `<=`, `>=` are lowered via `Comparable`.

---

### 7.4 Example

```bestie
class Vec2 impl Addable<Vec2> {
    var x: float64
    var y: float64
    fun add(other: Vec2): Vec2 = Vec2(x + other.x, y + other.y)
    fun addAssign(other: Vec2) { x += other.x; y += other.y }
}

val a = Vec2(1.0, 2.0)
val b = Vec2(3.0, 4.0)
val c = a + b    // compile-time lowered to a.add(b)
```

---

### 7.5 Rules

* Operator protocols are **opt-in** — no type is forced to implement them
* All dispatch is **static** — fully resolved at compile time
* Implementing an assign variant (`addAssign`) requires the type to be mutable
* `Self` in `Negatable` refers to the implementing type
* Mixed-type operators (e.g. `Vec2 + float64`) are supported by parameterizing `T` differently: `impl Addable<float64>`

---

## Summary

The utility package provides:

* Canonical utility for efficient string construction (`StringBuilder`)
* Explicit absence modeling (`option`)
* Typed failure (`result`)
* Structural equality (`Equable`)
* Ordering contracts (`Comparable`)
* Hash-based identity (`Hashable`)
* Operator overloading (`Addable`, `Subtractable`, `Multipliable`, `Divisible`, `Modulable`, `Negatable`, `Indexable`, `IndexAssignable`)

These utilities establish the rhythm of Bestie’s standard library: **explicit, orthogonal, and compiler-verifiable**.

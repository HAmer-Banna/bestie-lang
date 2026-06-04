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

val s = sb.toStr()
```

`toStr()` (not `toString()`) is used deliberately — it matches the universal `toStr()` conversion convention used by every core type (`core/types.md` §2.1). There is exactly one spelling for "produce a `str`."

### Relationship to `std-lib.strings`

`StringBuilder` and `std-lib.strings` are complementary and do **not** overlap:

| Concern | Owner |
| ------- | ----- |
| Mutable, allocation-efficient **construction** (append in a loop) | `StringBuilder` (this section) |
| Immutable **queries / transforms** on an existing `str` (parse, substring, split, trim, case, search) | `std-lib.strings` |

`StringBuilder` operates on a mutable buffer and exposes `append` / `toStr`; `std-lib.strings` operates on immutable `str` values and returns new `str`s. They share no method names and never compete for the same operation — build with `StringBuilder`, then query/transform the resulting `str` with `std-lib.strings`.

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

## 8. Copyable and DeepCopyable

Bestie distinguishes three separate operations. Conflating them is the source of most copy-related bugs in other languages, so they are kept explicit:

| Operation | Meaning |
| --------- | ------- |
| `val b = a` (binding) | Governed by the **type**: a **copy** for value types; for owning/reference types an explicit `move` or borrow is required (bare binding of an owning value is a compile error). **Never a hidden deep copy or implicit move.** |
| `copy(a)` | An explicit **shallow** independent duplicate |
| `deepCopy(a)` | An explicit **deep** duplicate of the entire owned subgraph |

### 8.1 Protocols

```bestie
protocol Copyable<T> {
    fun copy(): T          // shallow independent duplicate
}

protocol DeepCopyable<T> {
    fun deepCopy(): T      // deep independent duplicate of the owned subgraph
}
```

Free functions dispatch to these protocols (or to a compiler-derived default):

```bestie
fun copy<T>(value: T): T        // requires T : Copyable
fun deepCopy<T>(value: T): T    // requires T : DeepCopyable
```

**There is no separate `Cloneable` protocol, and there are no marker protocols in Bestie.** `Copyable` / `DeepCopyable` *are* Bestie's "clone" mechanism, and they are **method-bearing** contracts (`copy()` / `deepCopy()`) — not Java-style empty markers that rely on a magic `Object.clone()`. A type opts in by satisfying a real method (explicitly or by compiler derivation, §8.2); capability is expressed by the method that performs the work, never by a contentless tag. This keeps duplication explicit, statically resolved, and free of reflective or runtime cloning machinery.

### 8.2 Compiler Derivation

Like `Equable`, copy behavior is **compiler-derivable**, with no runtime reflection:

* **Value types** (`primitives`, `value class`, `data class`, `tuple`, `enum`) are trivially `Copyable` — a structural copy. For these, `copy` and `deepCopy` are identical because there is no owned subgraph.
* A **`class` with no `own` fields** auto-derives `Copyable`: a new heap object with fields copied shallowly. Any `ref` or `ptr<T>` fields are copied as-is (the duplicate **aliases** the same borrowed/raw targets).
* `DeepCopyable` auto-derives only when **every `own` field is itself `DeepCopyable`** — each owned field is recursively duplicated into a fresh allocation, producing a fully independent owning graph.
* A type may **`impl` either protocol manually** to override the default (e.g. to copy a cache lazily, or to deep-copy across a `ptr` boundary it knows the size of).

### 8.3 Shallow Copy and Ownership

A shallow `copy()` of a type that owns memory would duplicate the owning pointer — two owners, one allocation, an inevitable double free. Therefore:

* **`copy()` is forbidden on a type with `own` fields** — it is a compile-time error. Use `deepCopy()`, which clones the owned subgraph so each result owns its own copy.
* `copy()` **is** allowed when the only non-value fields are `ref` or `ptr<T>`, because those carry no ownership — the duplicate simply shares the same targets (explicit aliasing).

```bestie
class Cache {            // owns a buffer
    val own data: Buffer
}

val c2 = copy(c)         // ❌ compile error: Cache has an own field — use deepCopy
val c3 = deepCopy(c)     // ✅ new Cache owning a fresh copy of data
```

### 8.4 Raw Pointers Are Never Followed

Copying a `ptr<T>` duplicates the **address**, not the pointee — for both `copy` and `deepCopy`. A raw pointer carries no ownership or size information, so it is outside the managed graph by design:

```bestie
val p2 = copy(p)        // same address as p (aliasing)
val p3 = deepCopy(p)    // also the same address — deepCopy does not chase raw pointers
```

To duplicate what a `ptr<T>` points at, the programmer must do it explicitly (they alone know the size and lifetime).

### 8.5 Containers

| Element type | `copy()` | `deepCopy()` |
| ------------ | -------- | ------------ |
| `list<value T>` | new container, elements copied (independent) | identical to `copy()` |
| `list<own T>` | ❌ forbidden — would duplicate ownership | new container, **each element deep-copied** |
| `list<ref T>` | new container, **same borrows** (aliased) | identical (borrows aren't owned) |
| `list<ptr<T>>` | new container, **same addresses** (aliased) | identical (raw pointers not followed) |

This is consistent with §8.3–8.4: ownership is duplicated only by `deepCopy`, never silently.

### 8.6 Immutable Values

For an immutable value type such as `str`, `val b = a`, `copy(a)`, and `deepCopy(a)` are **observably identical** — each yields an independent value, and whether the backing storage is shared is an invisible implementation detail (safe precisely because the value cannot mutate). The copy/deep-copy distinction only becomes observable for **mutable** or **owning** types.

### 8.7 Copy Operates on Fields, Never Accessors

`copy()` and `deepCopy()` duplicate **stored fields directly**. They never invoke user methods — no `get()`, no resolver, no lazy-initialization trigger. This guarantees:

* **Laziness is preserved.** Copying an object that has not yet computed a cached or lazily-initialized field copies the *uncomputed* state; it does not force materialization.
* **No surprise side effects.** Duplication cannot run arbitrary user code through accessor methods.

This matters for indirection patterns such as `Proxy<T>` (`patterns.md` §5): copying a proxy copies its stored indirection state per the field rules above — it does **not** call `get()` and does **not** resolve the proxied target. Whether the target is duplicated depends solely on how the proxy holds it:

* proxy holds the target as an `own` field → `deepCopy` duplicates the target; `copy` is forbidden
* proxy holds it as `ref` / `ptr<T>` → both `copy` and `deepCopy` alias the same target (the target's lifetime remains the programmer's responsibility)

A type that genuinely needs copy to resolve or transform a field must `impl Copyable` / `DeepCopyable` **manually** and do so explicitly.

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
* Explicit duplication (`Copyable` / `copy`, `DeepCopyable` / `deepCopy`)

These utilities establish the rhythm of Bestie’s standard library: **explicit, orthogonal, and compiler-verifiable**.

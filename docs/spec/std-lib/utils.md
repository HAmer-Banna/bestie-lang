# Bestie Standard Library — Utilities (`bestie.lib.utils`)

This document defines the **utils package** of the Bestie standard library. It holds the protocols that core cites by name — iteration, equality, ordering, hashing, and the operator protocols — together with the small concrete utilities that do not belong to any other package.

Every symbol core depends on lives here, which is what makes `core/lang.md` §27's citation table point at a single package.

Bestie uses lowercase for foundational library types such as `set<T>` and `map<K,V>`, while nominal concrete utility types such as `StringBuilder` remain PascalCase.

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

### Relationship to `bestie.lib.strings`

`StringBuilder` and `bestie.lib.strings` are complementary and do **not** overlap:

| Concern | Owner |
| ------- | ----- |
| Mutable, allocation-efficient **construction** (append in a loop) | `StringBuilder` (this section) |
| Immutable **queries / transforms** on an existing `str` (parse, substring, split, trim, case, search) | `bestie.lib.strings` |

`StringBuilder` operates on a mutable buffer and exposes `append` / `toStr`; `bestie.lib.strings` operates on immutable `str` values and returns new `str`s. They share no method names and never compete for the same operation — build with `StringBuilder`, then query/transform the resulting `str` with `bestie.lib.strings`.

---

## 2. Iterator and Iterable

These two protocols are the ones `for/in` is defined against. `core/lang.md` §13 states the loop's desugaring in terms of `iterator()` and `next()`, and §27 lists both as **frozen**: they may not be renamed, nor have the shape core relies on changed, while the loop keyword exists. Everything else about them evolves under normal std-lib rules.

### 2.1 `Iterator<T>`

Explicit, pull-based iteration.

```bestie
protocol Iterator<T> {
    fun next(): T ?
}
```

Semantics:

* `next()` returns the next element, or **absent** when iteration is complete
* Iterators are **stateful**
* No implicit allocation, and no hidden invalidation rules

Rules:

* An iterator may own or borrow its data source
* Thread safety depends on the underlying object
* Iterators are not restartable unless explicitly documented

### 2.2 `Iterable<T>`

Things that can produce an iterator.

```bestie
protocol Iterable<T> {
    fun iterator(): Iterator<T>
}
```

Semantics:

* `iterator()` creates fresh iteration state
* No requirement for heap allocation
* Multiple iterators may coexist if the implementation allows it

A type does **not** have to implement this protocol to work with `for/in`: core requires only the *shape* — a `fun iterator()` whose result has a `fun next(): T ?`. Implementing `Iterable<T>` is the conventional way to have that shape, and is what core `array<T>`, `slice<T>`, `range<T>`, and every std-lib collection do.

### 2.3 Why this is the constraint for generic collection code

Collection variations are invariant (`std-lib/collections.md` §3.3), so a function that names `list<int>` accepts array-backed lists and nothing else. Generic code takes `Iterable<T>` instead:

```bestie
fun <C impl Iterable<int>> sum(xs: C): int {
    var t = 0
    for (x in xs) { t += x }
    return t
}
```

This is looser *and* more capable than a concrete parameter: it accepts every list variation plus `array<int>`, `slice<int>`, `set<int>`, and `range<int>`. It is fully monomorphized, so each instantiation compiles to the same code a hand-written loop would.

Name a concrete type only where the cost is part of the contract — `fun binarySearch(xs: list<int>, target: int): int ?` says array-backed because O(log n) depends on it.

---

## 3. Equable Protocol

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

If `Equable` is not implemented, core supplies the default: structural comparison for value types, identity comparison for reference types. The full per-kind table is in `core/lang.md` §15.4 — equality is core semantics, so this protocol overrides a defined default rather than filling a gap.

No dynamic dispatch is introduced.

---

## 4. Comparable Protocol

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

## 5. Hashable Protocol

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

## 6. Operator Overloading Protocols

Bestie supports operator overloading through **explicit protocols**. The compiler lowers operator expressions to protocol method calls at compile time — fully monomorphized, no runtime dispatch, no vtables.

All operator protocols are resolved at compile time. Using an operator on a type that does not implement the corresponding protocol is a **compile-time error**.

> **Core owns the lowering; this package owns the protocols.** The authoritative table of which operator lowers to which method is `core/lang.md` §15.3 — an operator's meaning is language syntax and cannot be redefined here. This package declares the protocols that supply those methods, and a type opts in by implementing one.
>
> Consequently the method names below — `add`, `sub`, `mul`, `div`, `mod`, `neg`, the `*Assign` forms, `get`, `set`, `equal`, `compareTo`, `hash` — are **cited symbols** (`core/lang.md` §27) and are frozen. New protocols and new members may be added here freely; these names may not be renamed or removed.

---

### 6.1 Arithmetic Operators

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

### 6.2 Index Operators

```bestie
protocol Indexable<I, T> {
    fun get(index: I): T        // a[i]
}

protocol IndexAssignable<I, T> {
    fun set(index: I, val: T)   // a[i] = v
}
```

---

### 6.3 Lowering Rules

The lowering table is **normative in `core/lang.md` §15.3**, which also covers the compound-assignment forms, the comparison lowering through `compareTo`, and the operators that are deliberately *not* overloadable (bitwise, logical, and the overflow-explicit `+%` / `+|` / `+!` family).

Summarized here for convenience:

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

`==` and `!=` lower via `Equable`; `<`, `>`, `<=`, `>=` lower via `Comparable`. When a type implements neither, core supplies a default: structural comparison for value types, identity for reference types (`core/lang.md` §15.4).

---

### 6.4 Example

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

### 6.5 Rules

* Operator protocols are **opt-in** — no type is forced to implement them
* All dispatch is **static** — fully resolved at compile time
* Implementing an assign variant (`addAssign`) requires the type to be mutable
* `Self` in `Negatable` refers to the implementing type
* Mixed-type operators (e.g. `Vec2 + float64`) are supported by parameterizing `T` differently: `impl Addable<float64>`

---

## 7. Copyable and DeepCopyable

Bestie distinguishes three separate operations. Conflating them is the source of most copy-related bugs in other languages, so they are kept explicit:

| Operation | Meaning |
| --------- | ------- |
| `val b = a` (binding) | Governed by the **type**: a **copy** for value types; for owning/reference types an explicit `move` or borrow is required (bare binding of an owning value is a compile error). **Never a hidden deep copy or implicit move.** |
| `copy(a)` | An explicit **shallow** independent duplicate |
| `deepCopy(a)` | An explicit **deep** duplicate of the entire owned subgraph |

### 7.1 Protocols

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

### 7.2 Compiler Derivation

Like `Equable`, copy behavior is **compiler-derivable**, with no runtime reflection:

* **Value types** (`primitives`, `value class`, `data class`, `tuple`, `enum`) are trivially `Copyable` — a structural copy. For these, `copy` and `deepCopy` are identical because there is no owned subgraph.
* A **`class` with no `own` fields** auto-derives `Copyable`: a new heap object with fields copied shallowly. Any `ref` or `ptr<T>` fields are copied as-is (the duplicate **aliases** the same borrowed/raw targets).
* `DeepCopyable` auto-derives only when **every `own` field is itself `DeepCopyable`** — each owned field is recursively duplicated into a fresh allocation, producing a fully independent owning graph.
* A type may **`impl` either protocol manually** to override the default (e.g. to copy a cache lazily, or to deep-copy across a `ptr` boundary it knows the size of).

### 7.3 Shallow Copy and Ownership

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

### 7.4 Raw Pointers Are Never Followed

Copying a `ptr<T>` duplicates the **address**, not the pointee — for both `copy` and `deepCopy`. A raw pointer carries no ownership or size information, so it is outside the managed graph by design:

```bestie
val p2 = copy(p)        // same address as p (aliasing)
val p3 = deepCopy(p)    // also the same address — deepCopy does not chase raw pointers
```

To duplicate what a `ptr<T>` points at, the programmer must do it explicitly (they alone know the size and lifetime).

### 7.5 Containers

| Element type | `copy()` | `deepCopy()` |
| ------------ | -------- | ------------ |
| `list<value T>` | new container, elements copied (independent) | identical to `copy()` |
| `list<own T>` | ❌ forbidden — would duplicate ownership | new container, **each element deep-copied** |
| `list<ref T>` | new container, **same handles** (aliased; list does not own) | identical (handles aren't owned) |
| `list<ptr<T>>` | new container, **same addresses** (aliased) | identical (raw pointers not followed) |

This is consistent with §8.3–8.4: ownership is duplicated only by `deepCopy`, never silently.

### 7.6 Immutable Values

For an immutable value type such as `str`, `val b = a`, `copy(a)`, and `deepCopy(a)` are **observably identical** — each yields an independent value, and whether the backing storage is shared is an invisible implementation detail (safe precisely because the value cannot mutate). The copy/deep-copy distinction only becomes observable for **mutable** or **owning** types.

### 7.7 Copy Operates on Fields, Never Accessors

`copy()` and `deepCopy()` duplicate **stored fields directly**. They never invoke user methods — no `get()`, no resolver, no lazy-initialization trigger. This guarantees:

* **Laziness is preserved.** Copying an object that has not yet computed a cached or lazily-initialized field copies the *uncomputed* state; it does not force materialization.
* **No surprise side effects.** Duplication cannot run arbitrary user code through accessor methods.

This matters for any hand-written indirection type — a lazy handle, a cache entry, a wrapper around a `ptr<T>`. Copying one copies its stored state per the field rules above; it does **not** call an accessor and does **not** resolve whatever the indirection points at. Whether the target is duplicated depends solely on how the field is qualified:

* target held as an `own` field → `deepCopy` duplicates it; `copy` is forbidden
* target held as `ref` / `ptr<T>` → both `copy` and `deepCopy` alias the same target (its lifetime remains the programmer's responsibility)

A type that genuinely needs copy to resolve or transform a field must `impl Copyable` / `DeepCopyable` **manually** and do so explicitly.

---

## Summary

The utility package provides:

* Canonical utility for efficient string construction (`StringBuilder`)
* Iteration contracts (`Iterator`, `Iterable`) — cited by core's `for/in`
* Structural equality (`Equable`)
* Ordering contracts (`Comparable`)
* Hash-based identity (`Hashable`)
* Operator overloading (`Addable`, `Subtractable`, `Multipliable`, `Divisible`, `Modulable`, `Negatable`, `Indexable`, `IndexAssignable`)
* Explicit duplication (`Copyable` / `copy`, `DeepCopyable` / `deepCopy`)

These utilities establish the rhythm of Bestie’s standard library: **explicit, orthogonal, and compiler-verifiable**.

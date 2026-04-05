# Bestie Standard Library — Collections

This document defines the **Bestie collections** provided by the standard library.
Collections are **generic, type-safe, deterministic, and ownership-aware**, with **explicit construction rules** and **compile-time guarantees**.

Core Bestie includes exactly one built-in collection: `list<T>`.
The full `list<T>` model is defined by the core language and is not specified in this document.
This document covers the standard-library collection layer: `set`, `map`, `deque`, and `heap`.

---

## 1. Supported Collections

| Collection | Variations         | Default |
| ---------- | ------------------ | ------- |
| `set<T>`   | hash, tree, linked | hash    |
| `map<K,V>` | hash, tree, linked | hash    |
| `deque<T>` | queue, stack       | as-is   |
| `heap<T>`  | max, min           | ❌ none  |

All collections are **generic** (`<T>`).
All variations are **explicit** and **compile-time validated**.

Collection family names stay lowercase across Bestie so they remain aligned with core `list<T>` and other foundational abstractions such as `option<T>` and `result<T,E>`.

---

## 2. Construction Model

Collections are created using **builders**, **size annotations**, or **literals**, depending on intent.

There is no hidden allocation strategy.
Growth/reallocation behavior is defined by the selected collection variation.

---

### 2.1 Builder Construction

```bestie
val ys = set<int>.tree().add(1).add(2)
val zs = map<int,str>.hash().put(1,"a").put(2,"b")
val dq = deque<int>.queue().build()
```

Rules:

* Builders are explicit
* Builder chains are resolved at compile time
* Allocation strategy is known before code generation

---

### 2.2 Core `list<T>` Boundary

`list<T>` is entirely part of the core language.
This includes:

* The existence of `list<T>`
* The available list variations
* `list<T>.build()`
* List literals
* Sized array forms such as `list<T>[n]`
* Bracket validity rules for array-backed lists
* List methods
* List immutability and concurrency modifiers

`bestie.lib.collections` does not define or extend `list<T>`.

---

## 3. Defaults and Variations

### 3.1 Defaults

* `set<T>` → hash
* `map<K,V>` → hash
* `deque<T>` → must explicitly choose queue or stack behavior
* `heap<T>` → **must specify `max` or `min`**

```bestie
val xs = set<int>.build()                   // hash-backed
val ys = map<int,str>.build()               // hash-backed
val dq = deque<int>.queue().build()         // queue
```

---

### 3.2 Conflicting Variations

If multiple variations from the **same category** are chained, the **last one wins**.

```bestie
val x = set<int>.hash.tree().build()       // tree
val y = map<int,str>.linked.hash().build() // hash
```

Conflicts across incompatible categories are **compile-time errors**.

---

## 4. Immutability and Concurrency

Collections support explicit mutation and concurrency semantics.

### 4.1 Mutation Semantics

* `mutable` — default
* `immutable` — value-based, mutation creates a new collection
* `copyOnWrite` — lazy copying on mutation

### 4.2 Concurrency Semantics

* `concurrent` — thread-safe mutable access with defined guarantees

### 4.3 Resolution Rules

* `immutable` **dominates all other mutation or concurrency modifiers**
* `concurrent` has no effect on immutable collections
* `copyOnWrite` is ignored if `immutable` is present

```bestie
set<int>.concurrent.copyOnWrite.immutable.build()
```

Resolves to:

```text
immutable set
```

Optional compiler warnings may be emitted for redundant modifiers.

---

## 5. Collection Literals and `const`

Collection literals use `{}` syntax and are compile-time parsed with explicit runtime allocation semantics.

```bestie
val a : set<int> = {1,2,3}
val b : map<str,int> = {"a": 1, "b": 2}
```

### 5.1 Literal Rules

* `set` literals reject duplicates when duplicates are compile-time provable
* `map` literals reject duplicate keys when duplicates are compile-time provable
* No automatic deduplication is performed

```bestie
val x : set<int> = {1,2,2} // ❌ compile-time error
val y : map<str,int> = {"a": 1, "a": 2} // ❌ compile-time error
```

---

### 5.2 `const` Collections

A collection may be declared `const` **only if** it is created via a literal.

```bestie
const xs : set<int> = {1,2,3}
const ys : map<str,int> = {"x": 1}
```

Invalid:

```bestie
const a = set<int>.build()        // ❌ runtime allocation
const b = map<str,int>.build()    // ❌ runtime allocation
```

`const` collections:

* Are fully immutable
* Have no heap allocation
* Reside in read-only memory
* Cannot be mutated
* Preserve immutability when copied

---

## 6. Collection Methods

All collections use **consistent method naming**.

| Collection | Methods                                                           |
| ---------- | ----------------------------------------------------------------- |
| set        | `add`, `remove`, `contains`                                       |
| heap       | `add`, `remove`, `peek`                                           |
| deque      | `addFirst`, `addLast`, `removeFirst`, `removeLast`, `peekFirst`, `peekLast` |
| map        | `put`, `get`, `remove`, `containsKey`                             |

All collections provide **efficient iterators** with no hidden indirection.
Element access syntax is explicit (for example: `map["food"]`).

---

## 7. Functional Operations

Std-lib collections **do not include functional methods** (`map`, `filter`, `fold`).

Functional behavior is provided by:

* `std.functional`
* Extension functions

```bestie
val sum = std.functional.sum(xs)
```

This keeps collections minimal and zero-cost.

---

## 8. Relationship to Iterable
* All collections `impl Iterable<T>`
* `Iterable<T>` does not imply mutability
* Iteration order is defined by the implementation
* `for-in` is preferred over direct `.next()` usage in user code

---

## 9. Summary

Bestie collections are:

* Generic and type-safe
* Explicitly constructed
* Deterministic in memory layout
* Ownership-aware
* Compile-time validated
* Free of hidden allocation behavior
* Explicitly immutable via `immutable` or `const`

> Collections in Bestie are **explicit tools**, not abstractions with surprises — matching the language's core philosophy of predictability, safety, and control.

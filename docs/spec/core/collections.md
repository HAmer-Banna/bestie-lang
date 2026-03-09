# Bestie Core Collections

This document defines the **core collections** in Bestie.
Collections are **generic, type-safe, deterministic, and ownership-aware**, with **explicit construction rules** and **compile-time guarantees**.

Collections are value-oriented by default and integrate tightly with Bestie’s memory and concurrency model.

---

## 1. Supported Core Collections

| Collection | Variations         | Default |
| ---------- | ------------------ | ------- |
| `list<T>`  | array, linked      | array   |
| `set<T>`   | hash, tree, linked | hash    |
| `map<K,V>` | hash, tree, linked | hash    |
| `deque<T>` | queue, stack       | as-is   |
| `heap<T>`  | max, min           | ❌ none  |

All collections are **generic** (`<T>`).
All variations are **explicit** and **compile-time validated**.

---

## 2. Construction Model

Collections are created using **builders**, **size annotations**, or **literals**, depending on intent.

There is no implicit allocation and no hidden resizing.

---

### 2.1 Builder Construction

```bestie
val xs = list<int>.build()
val ys = set<int>.tree().add(1).add(2)
val zs = map<int,str>.hash().put(1,"a").put(2,"b")
```

Rules:

* Builders are explicit
* Builder chains are resolved at compile time
* Allocation strategy is known before code generation

---

### 2.2 Sized Arrays (No Matrix Type)

Bestie does **not** provide a separate `matrix` collection.

Instead, **fixed-size and multi-dimensional arrays** are expressed using **bracket syntax**, applicable **only to array-backed lists**.

#### Fixed-size array

```bestie
val a = list<int>[10].build()
```

#### Dynamic-size array

```bestie
val n = readInt()
val b = list<int>[n].build()
```

#### Two-dimensional array (flat, C-style layout)

```bestie
val m = list<int>[2][3].build()
```

Properties:

* `[n]`, `[r][c]` are part of the **type**
* Memory is **contiguous and flat**
* Indexing is row-major
* No matrix object exists in the core language

---

### 2.3 Validity Rules for Brackets

Brackets are valid **only** for array-backed lists.

| Expression             | Result               |
| ---------------------- | -------------------- |
| `list<int>[10]`        | ✅ valid              |
| `list<int>.array[10]`  | ✅ valid              |
| `list<int>.linked[10]` | ❌ compile-time error |
| `set<int>[10]`         | ❌ compile-time error |

This rule is strict and enforced by the compiler.

---

## 3. Defaults and Variations

### 3.1 Defaults

* `list<T>` → array
* `set<T>` → hash
* `map<K,V>` → hash
* `deque<T>` → as declared
* `heap<T>` → **must specify `max` or `min`**

```bestie
val xs : list<int> = list<int>.build()  // array-backed
```

---

### 3.2 Conflicting Variations

If multiple variations from the **same category** are chained, the **last one wins**.

```bestie
val x = list<int>.array.linked().build()   // linked
val y = set<int>.hash.tree().build()       // tree
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
list<int>.concurrent.copyOnWrite.immutable.build()
```

Resolves to:

```text
immutable list
```

Optional compiler warnings may be emitted for redundant modifiers.

---

## 5. Collection Literals and `const`

Collection literals use `{}` syntax and are **fully compile-time constructs**.

```bestie
val a : list<int> = {1,2,3}
val b : set<int>  = {1,2,3}
```

### 5.1 Literal Rules

* `list` literals allow duplicates
* `set` literals **reject duplicates at compile time**
* No automatic deduplication is performed

```bestie
val x : set<int> = {1,2,2} // ❌ compile-time error
```

---

### 5.2 `const` Collections

A collection may be declared `const` **only if** it is created via a literal.

```bestie
const xs : list<int> = {1,2,3}
const ys : list<int>[2][2] = {1,0,0,1}
```

Invalid:

```bestie
const a = list<int>.build()        // ❌ runtime allocation
const b = list<int>[n].build()    // ❌ runtime size
```

`const` collections:

* Are fully immutable
* Have no heap allocation
* Reside in read-only memory
* Cannot be mutated or copied

---

## 6. Collection Methods

All collections use **consistent method naming**.

| Collection    | Methods                                            |
| ------------- | -------------------------------------------------- |
| list/set      | `add`, `remove`, `get`, `insert`, `indexOf`                                 |
| heap          | `add`, `remove`, `get`
| deque         | `addFirst`, `addLast`, `removeFirst`, `removeLast`, `peekFirst`, `peekLast` |
| map           | `put`, `remove`                                    |

All collections provide **efficient iterators** with no hidden indirection.
Some collections provide convenient way for accessing an element; as list[0]; map["food"];

---

## 7. Functional Operations

Core collections **do not include functional methods** (`map`, `filter`, `fold`).

Functional behavior is provided by:

* `std-lib functional`
* Extension functions

```bestie
val sum = list<int>.of(1,2,3).sum()
```

This keeps the core minimal and zero-cost.

---

## 8. Relationship to Iterable
* All core collections implement Iterable<T>
* Iterable<T> does not imply mutability
* Iteration order is defined by the implementation
* for-in loop is used as replacement for .next()

---

## 9. Summary

Bestie collections are:

* Generic and type-safe
* Explicitly constructed
* Deterministic in memory layout
* Ownership-aware
* Compile-time validated
* Free of hidden allocation or resizing
* Immutable-by-design when declared as values or constants

> Collections in Bestie are **explicit tools**, not abstractions with surprises — matching the language’s core philosophy of predictability, safety, and control.

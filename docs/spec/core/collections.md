# Bestie Core Collections

This document defines the **core collections** in Bestie. Collections are **generic, type-safe, deterministic, and ownership-aware**, constructed using the **builder/factory pattern**.

---

## 1. Supported Core Collections

| Collection | Variations / Views    | Default                          |
| ---------- | --------------------- | -------------------------------- |
| `list<T>`  | array, matrix, linked | array                            |
| `set<T>`   | tree, hash, linked    | hash                             |
| `map<K,V>` | tree, hash, linked    | hash                             |
| `deque<T>` | queue, stack          | as-is                            |
| `heap<T>`  | max, min              | ❌ No default — user must specify |

All collections are **generic** (`<T>`). Builders provide a consistent, fluent construction pattern.

---

## 2. Builder Pattern & Construction

Collections are created using **builders/factories**:

```bestie
val xs : list<int> = list<int>.of(1,2,3)
val ys = set<int>.tree().add(1).add(2)
val zs = map<int,str>.hash().put(1,"a").put(2,"b")
```

### 2.1 Defaults

* `list.array`
* `list.matrix` (C-style, flat memory layout)
* `list.linked`
* `set.hash`
* `map.hash`
* `deque` (as-is)
* `heap` must specify `max` or `min`

```bestie
val xs : list<int> = list<int>.array().build() // same as list<int>.build()
```

### 2.2 Conflicting Variations

If multiple variations are chained **within the same category**, the **last variation wins**:

```bestie
val x = list<int>.array.linked()  // equivalent to list<int>.linked
val y = list<int>.concurrent.copyOnWrite() // equivalent to copyOnWrite
```

---

### 2.3 Immutability & Concurrency

* `immutable()` — returns a copy-on-write collection; mutation creates a new collection
* `concurrent()` — provides thread-safe iteration with weaker guarantees
* `copyOnWrite()` — optional specialized builder for high concurrency

---

## 3. Collection Literals

Literals use `{}` syntax. Type is resolved by compiler:

```bestie
val x : list<int> = {1,2,3,3} // ✅ duplicates allowed in list
val y : set<int> = {1,2,3}    // ✅ unique
val z : set<int> = {1,2,2,3}  // ❌ compiler error: duplicate elements
```

* **Set duplicates** are **not automatically removed**. Developers must resolve duplicates; IDE can suggest fixes.
* Type must **match declared collection variant**:

```bestie
val x : list<int>.linked = list<int>.of(1,2,3) // ❌ error
```

Use either the variant in declaration or let compiler infer type.

---

## 4. Collection Methods

All collections share **consistent method names**:

| Collection    | Methods                                            |
| ------------- | -------------------------------------------------- |
| list/set/heap | `add(element)`, `remove(element)`                  |
| deque         | `addFirst`, `addLast`, `removeFirst`, `removeLast` |
| map           | `put(key,value)`, `remove(key)`                    |

All collections implement **efficient iterators**. No `Map.Entry` indirection like Java.

---

## 5. Matrix

`list.matrix` is **C-style (flat memory layout)**:

* Stored in a **single contiguous block**
* Row/column access via helper methods (`matrix[row, col]`)
* Fully compatible with **Arena allocation**
* Provides predictable memory layout and high performance

> Advanced linear algebra operations are in the **std-lib math package**, where matrices are treated as first-class objects with linear algebra support.

---

## 6. Variations & Views

### List Variations

* `array` — default, contiguous memory
* `matrix` — C-style flat matrix
* `linked` — linked list

### Set Variations

* `hash` — default
* `tree` — ordered
* `linked` — insertion-ordered

### Map Variations

* `hash` — default
* `tree` — ordered
* `linked` — insertion-ordered

### Deque Variations

* `queue`
* `stack`

### Heap Variations

* `max` — maximum at root
* `min` — minimum at root

---

## 7. Functional Methods

Collections **do not provide built-in functional methods** (map, filter, fold).
Use the **std-lib functional package** or **extension functions**:

```bestie
val sum = list<int>.of(1,2,3).sum()  // sum as extension
```

---

## 8. Summary

Bestie collections are:

* Generic, type-safe, and consistent
* Built using **builder/factory patterns**
* Deterministic memory layout and ownership-safe
* Immutable and concurrent variants available
* Conflicting builder variations resolve to the last variation
* Minimal in the core, expressive through std-lib and extensions

> Collections in Bestie are **predictable and rhythmic**, matching the language’s philosophy: explicit, safe, and consistent.

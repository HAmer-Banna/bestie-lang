# Bestie Standard Library — Collections

This document defines the **Bestie collections** provided by the standard library.
Collections are **generic, type-safe, deterministic, and ownership-aware**, with **explicit construction rules** and **compile-time guarantees**.

Core Bestie provides `array<T>` — a fixed-capacity built-in collection defined in `core/types.md`.
This document covers the standard-library collection layer: `list<T>`, `set`, `map`, `deque`, and `heap`.

---

## 0. Class Kind and Ownership Rationale

### Class kind

All five standard-library collection types — `list<T>`, `set<T>`, `map<K,V>`, `deque<T>`, `heap<T>` — are **`class`**.

They are not `value class` or `data class` because:

* They have **mutable internal state** by default (element count, backing buffer, tree/bucket structure)
* They have **identity** — two collections containing the same elements are equal in content but are distinct objects in memory
* They **own their backing storage** — the internal buffer or node pool is heap-allocated and freed when the collection is freed

The `.immutable` modifier produces a variant that exposes no mutation API, but the underlying type is still a `class`.

### Field-level ownership for element types

A collection's ownership of its elements is determined by the element type's `own` qualifier:

| Declaration | Meaning |
| ----------- | ------- |
| `set<User>` | The set holds **copies** of `User` values. `User` is a value type. |
| `set<own User>` | The set **owns** each `User` instance. `freeDeep()` frees all elements. |
| `set<ref User>` | The set **borrows** `User` instances. It is not responsible for freeing them. |

The same rule applies to `list<T>`, `map<K,V>`, `deque<T>`, and `heap<T>` for both key and value types where applicable.

---

## 1. Supported Collections

| Collection | Variations                        | Default       |
| ---------- | --------------------------------- | ------------- |
| `list<T>`  | linked (more sequence types TBD)  | array-backed  |
| `set<T>`   | hash, tree, linked                | hash          |
| `map<K,V>` | hash, tree, linked                | hash          |
| `deque<T>` | queue, stack                      | as-is         |
| `heap<T>`  | max, min                          | ❌ none        |

All collections are **generic** (`<T>`).
All variations are **explicit** and **compile-time validated**.

Collection family names stay lowercase across Bestie so they remain aligned with core `array<T>` and other foundational abstractions such as `option<T>` and `result<T,E>`.

`list<T>` is the primary dynamic collection. Its default (no variation keyword) is an array-backed resizable list. The `linked` variation selects a linked-list representation. Additional sequence variations may be introduced in future versions.

---

## 2. Construction Model

Collections are created using **builders**, **`of()` construction**, **size annotations**, or **literals**, depending on intent.

There is no hidden allocation strategy.
Growth/reallocation behavior is defined by the selected collection variation.

---

### 2.1 Builder Construction

```bestie
val ys = set<int>.tree.build().add(1).add(2)
val zs = map<int,str>.hash.build().put(1,"a").put(2,"b")
val dq = deque<int>.queue.build()
```

Rules:

* Builders are explicit
* Builder chains are resolved at compile time
* Allocation strategy is known before code generation

---

### 2.2 `of()` Construction

Collections may also be created directly from explicit values:

```bestie
val xs = set<int>.of(1, 2, 3)
val ys = deque<int>.queue.of(1, 2, 3)
```

Rules:

* `of()` is explicit construction, not hidden conversion
* Element values are visible at the call site
* The backing variation remains explicit when the collection family requires it

---

### 2.3 `list<T>` Construction

`list<T>` is a dynamic collection and is part of `bestie.lib.collections`.

```bestie
val xs = list<int>.build()               // empty array-backed dynamic list
val ys = list<int>.of(1, 2, 3, 4, 5)    // from explicit values
val ls : list<int> = {1, 2, 3}          // list literal
val ll = list<int>.linked.build()        // empty linked-list variant
```

Rules:

* `list<T>.build()` with no variation produces an array-backed resizable list
* `list<T>.linked.build()` produces a linked-list backed list
* `of()` is explicit construction; element values are visible at the call site
* List literals `{...}` are valid when the target type is `list<T>`

---

## 2a. `list<T>` — Dynamic Collection

`list<T>` is Bestie's dynamic, resizable collection.
It grows automatically as elements are added and shares an indexing interface with `array<T>`.

### 2a.1 Operators

| Operation | Meaning |
| --------- | ------- |
| `ls[i]` | Element access — **panics** if index is out of bounds |
| `ls[-i]` | Access from the end — `ls[-1]` is the last element. See §8a |
| `ls[i] = v` | Element assignment when mutable — **panics** if out of bounds |
| `ls[lo..hi]` | Slice — borrowed `slice<T>` view (array-backed only, no copy). See §8a |
| `for (x in ls)` | Iterate over all elements |

Out-of-bounds access is always a **panic** — a violated invariant with no recovery path.

Indexing, negative indexing, and slicing share one convention with core `array<T>` / `slice<T>`; the full rules and per-collection support table are in §8a.

### 2a.2 Methods

| Method | Returns | Notes |
| ------ | ------- | ----- |
| `add(value: T)` | `void` | Appends element; grows automatically |
| `insert(index: int, value: T)` | `void` | Inserts at position |
| `get(index: int)` | `T` | Explicit element access — **panics** if out of bounds |
| `remove(index: int)` | `T` | Removes and returns element |
| `indexOf(value: T)` | `int ?` | Returns index or absent |
| `size()` | `int` | Current number of elements |
| `isEmpty()` | `bool` | True when no elements exist |
| `toArray()` | `array<T>` | New fixed-capacity array with exactly `size()` slots, filled with the list's current elements |
| `toSet()` | `set<T>` | New hash set containing all elements (deduplicates) |

### 2a.3 Variations

| Variation | Meaning |
| --------- | ------- |
| _(none)_  | Array-backed resizable list — default |
| `linked`  | Doubly-linked list representation |

Future sequence variations may be introduced under the same builder-chain model.

### 2a.4 Shared Interface with `array<T>`

`list<T>` and `array<T>` (core) intentionally share the same indexing syntax and method names where they overlap:

```bestie
val arr : array<int>[] = {10, 20, 30}
val ls  : list<int>    = {10, 20, 30}

arr[0]      // 10
ls[0]       // 10 — identical syntax

arr.size()  // 3
ls.size()   // 3 — identical method

arr.add(40) // panics — at capacity
ls.add(40)  // fine  — list grows
```

The key distinction: `array<T>` is **static** (fixed capacity, panics on overflow); `list<T>` is **dynamic** (grows as needed, never overflows on add).

---

## 3. Defaults and Variations

### 3.1 Defaults

* `list<T>` → array-backed resizable
* `set<T>` → hash
* `map<K,V>` → hash
* `deque<T>` → must explicitly choose queue or stack behavior
* `heap<T>` → **must specify `max` or `min`**

```bestie
val ls = list<int>.build()                  // array-backed dynamic list
val xs = set<int>.build()                   // hash-backed
val ys = map<int,str>.build()               // hash-backed
val dq = deque<int>.queue.build()           // queue
```

---

### 3.2 Conflicting Variations

If multiple variations from the **same category** are chained, the **last one wins**.

```bestie
val x = set<int>.hash.tree.build()         // tree
val y = map<int,str>.linked.hash.build()   // hash
```

Conflicts across incompatible categories are **compile-time errors**.

---

## 4. Immutability and Concurrency

Collections support explicit mutation and concurrency semantics.
Bestie targets deterministic memory layout for collections **whenever the chosen representation permits it**.

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

### 4.4 `.immutable` Modifier vs `@immutable val`

These are two distinct mechanisms with fundamentally different semantics.

**`.immutable` modifier** — a collection variation. The collection exists in a functionally immutable state: mutation methods return a **new collection** rather than modifying in place. The original is always unchanged. This is the same model `str` uses — `+` on a `str` produces a new string, it does not modify the original:

```bestie
val ls = list<int>.immutable.of(1, 2, 3)
val ls2 = ls.add(4)     // ls  → still {1, 2, 3}
                        // ls2 → new list {1, 2, 3, 4}
```

The binding `ls` itself can be rebound if declared with `var`.

**`@immutable val`** — a compiler-enforced freeze on the binding. Any mutation attempt — including calls that would return a new collection — is a **compile-time error**. The binding can never be changed:

```bestie
@immutable val ls : list<int> = {1, 2, 3}
ls.add(4)       // ❌ compile-time error: mutation on @immutable binding
val ls2 = ls    // ✅ reading and copying are fine
```

Comparison:

| Mechanism | Mutation call behaviour | Produces new collection? | Rebind the variable? |
| --------- | ----------------------- | ------------------------ | -------------------- |
| `list<T>.immutable` | Returns new collection | ✅ allowed | depends on `val`/`var` |
| `@immutable val` | Compile-time error | ❌ forbidden | ❌ always |
| `const` (literal only) | Compile-time error | ❌ forbidden | ❌ always |

**The same distinction applies to `str`:**

`str` is already functionally immutable by nature — `+` and all transformation methods return a new `str`. Annotating a `str` binding with `@immutable val` goes one step further: it prevents even the production of new values from that binding:

```bestie
val s : str = "hello"
val s2 = s + " world"   // ✅ s unchanged, s2 is a new str

@immutable val s3 : str = "hello"
val s4 = s3 + " world"  // ❌ compile-time error: transformation on @immutable binding
```

Use `.immutable` when you want **persistent / functional collection behaviour** — safe sharing, history, pure functions.
Use `@immutable val` when any mutation attempt, however indirect, is a **programming error** that should never compile.

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

| Collection | Core methods                                                               |
| ---------- | -------------------------------------------------------------------------- |
| list       | `add`, `insert`, `get`, `remove`, `indexOf`, `size`, `isEmpty`             |
| set        | `add`, `remove`, `contains`                                                |
| heap       | `add`, `remove`, `peek`                                                    |
| deque      | `addFirst`, `addLast`, `removeFirst`, `removeLast`, `peekFirst`, `peekLast` |
| map        | `put`, `get`, `remove`, `containsKey`                                      |

All collections provide **efficient iterators** with no hidden indirection beyond the selected collection representation.
Element access syntax is explicit (for example: `prices["food"]`, `ls[3]`).

Map view methods (returns live views, not copies):

| Method | Returns | Notes |
| ------ | ------- | ----- |
| `map.keys()` | `set<K>` | The set of all keys |
| `map.values()` | `list<V>` | All values in iteration order |
| `map.entries()` | `list<(K, V)>` | All key-value pairs in iteration order |

---

## 7. Functional Operations

Std-lib collections **do not include functional methods** (`map`, `filter`, `fold`).

Functional behavior is provided by:

* `bestie.lib.functional`
* Extension functions

```bestie
val sum = bestie.lib.functional.fold(xs, 0, (acc: int, x: int) => acc + x)
```

This keeps collections minimal and zero-cost.

---

## 8. `for/in` and `Iterable<T>`

All collections and `array<T>` (core) implement `Iterable<T>`. The `for/in` loop works with every one of them naturally — no conversion, no import, no adapter required.

```bestie
val arr  : array<int>[]  = {1, 2, 3}
val ls   : list<int>     = {4, 5, 6}
val xs   : set<str>      = {"a", "b", "c"}
val dq   : deque<int>    = deque<int>.queue.of(7, 8, 9)

for (n in arr) { print(n.toStr()) }
for (n in ls)  { print(n.toStr()) }
for (s in xs)  { print(s) }
for (n in dq)  { print(n.toStr()) }
```

### Map iteration

`map<K,V>` iterates over `(K, V)` entry pairs. Use tuple destructuring in the `for` header:

```bestie
val prices : map<str, int> = {"apple": 3, "bread": 5}

for ((item, price) in prices) {
    print(item + " costs " + price.toStr())
}
```

To iterate keys or values only, use the view methods:

```bestie
for (k in prices.keys())   { print(k) }
for (v in prices.values()) { print(v.toStr()) }
```

### Rules

* `Iterable<T>` does not imply mutability — iterating never modifies the collection
* Iteration order is defined by the representation: insertion order for linked variants, unspecified for hash-backed, sorted for tree-backed
* `for/in` is preferred over direct `.next()` calls in user code
* `array<T>` iterates in index order, from `0` to `size() - 1`

---

## 8a. Indexing, Negative Indexing, and Slicing

Collections reuse the **same indexing and slicing surface** defined for `array<T>` and `slice<T>` in `core/types.md` (§5–§6). There is one convention across the language — `array<T>` and array-backed `list<T>` behave identically — but a collection only exposes what its representation can do **without hidden cost**.

### 8a.1 What each collection supports

| Collection | `c[i]` positional | `c[-i]` from end | `c[lo..hi]` slice |
| ---------- | ----------------- | ---------------- | ----------------- |
| `list<T>` (array-backed) | ✅ O(1) | ✅ O(1) | ✅ → `slice<T>` view (no copy) |
| `list<T>.linked` | ✅ O(n) | ✅ O(n) | ❌ no view — see §8a.5 |
| `deque<T>` | ends O(1); middle representation-dependent | ✅ `deq[-1]` O(1) | ❌ no view — see §8a.5 |
| `array<T>` (core) | ✅ O(1) | ✅ O(1) | ✅ → `slice<T>` view |
| `set<T>` | ❌ not positional | ❌ | ❌ |
| `map<K,V>` | `m[key]` is **key** access, not positional | ❌ | ❌ |

### 8a.2 Negative indexing

`c[-i]` counts from the end and is sugar for `c[c.size() - i]` — `c[-1]` is the last element. Its cost is exactly the cost of positional access on that representation (O(1) for array-backed `list`/`deque` ends, O(n) for `list.linked`, same as `c[i]`). Out-of-range negative indices **panic**, identical to positive access.

```bestie
val ls : list<int> = {10, 20, 30, 40}
ls[-1]    // 40
ls[-2]    // 30
```

### 8a.3 `deque[-1]` and the end accessors

For a `deque<T>`, end indexing is defined in terms of the existing peek methods — same element, different empty-behavior:

```bestie
val dq : deque<int> = deque<int>.queue.of(10, 20, 30)

dq[-1]            // 30  — equivalent element to dq.peekLast()
dq[0]             // 10  — equivalent element to dq.peekFirst()
```

| Expression | Returns | On empty deque |
| ---------- | ------- | -------------- |
| `dq[-1]` | `T` | **panics** (like all `[]` access) |
| `dq.peekLast()` | `T ?` | `option.Not_Present` |
| `dq[0]` | `T` | **panics** |
| `dq.peekFirst()` | `T ?` | `option.Not_Present` |

So `dq[-1]` and `dq.peekLast()` read the **same element**; they differ only at the boundary. Use `dq[-1]` when a present element is an invariant (you *expect* it to be there) and use `dq.peekLast()` when emptiness is an expected, recoverable case.

### 8a.4 Slicing collections

Slicing follows the core `slice<T>` rules (`core/types.md` §6) and only applies to **contiguous** storage:

```bestie
val ls : list<int> = {10, 20, 30, 40, 50}

val view : slice<int> = ls[1..4]   // borrowed view over {20, 30, 40} — no copy
val tail               = ls[-2..]  // {40, 50}
val copy : list<int>   = ls[1..4].toList()   // explicit O(n) owned copy
```

* Array-backed `list<T>` yields a `slice<T>` borrowing the list's buffer. While that slice is alive the list **cannot grow/reallocate** (borrow rule) — the compiler enforces this, preventing a dangling view.
* A mutable list sliced as `slice<var T>` allows `view[i] = v` to write through to the list.

### 8a.5 Edge cases — when you must use methods, not `[]`

The `[lo..hi]` view and positional `[]` are not available everywhere. In these cases you use accessor/conversion methods instead:

1. **Empty-safe access.** `c[i]`, `c[-1]`, and `get(index)` **panic** out of bounds. When absence is expected, use the option-returning accessors: `deque.peekFirst()` / `deque.peekLast()` return `T ?`, `list.indexOf(value)` returns `int ?`, and `map.get(key)` returns the value option. These never panic.

2. **`set<T>` has no positional index.** A set has no stable position, so `xs[2]` does not exist. Use `contains(value)`, or iterate with `for/in`. There is no slicing.

3. **`map<K,V>` indexing is key access, not positional.** `m[k]` looks up the value for key `k`. For a `map<int, V>`, `m[-1]` is a lookup for the key `-1` — **not** "the last entry." Positional and end-relative indexing therefore do not apply to maps; use `keys()` / `values()` / `entries()` views (and, for ordered variants, the first/last-entry accessors covered with the sequenced design) to reach a position.

4. **Unordered (hash) variants have no "first/last/from-end."** `set<T>` (hash) and `map<K,V>` (hash) have no defined order, so end-relative or positional access is meaningless. Use an ordered variant (`linked` / `tree`) when order matters.

5. **Non-contiguous storage cannot produce a slice view.** `list<T>.linked`, `deque<T>`, `set<T>`, and `map<K,V>` are not contiguous, so `c[lo..hi]` would have to copy O(n). Bestie does **not** hide that cost behind slice syntax — slicing is unavailable on them. To get a sub-range, copy explicitly (`.toList()` then slice the result) or traverse it as a sequence. (A zero-copy lazy traversal layer is being designed separately.)

6. **Searching by value, not position.** To find *where* a value is, use `list.indexOf(value): int ?` — `[]` is for when you already know the index.

---

## 9. Conversions

Conversions between collections are **explicit method calls**.
No implicit coercion between collection types ever occurs.
Every conversion method allocates a new collection.

### 9.1 Conversion Table

| Source | Method | Target | Notes |
| ------ | ------ | ------ | ----- |
| `array<T>` | `toList()` | `list<T>` | New array-backed list |
| `array<T>` | `toSet()` | `set<T>` | New hash set; deduplicates |
| `list<T>` | `toArray()` | `array<T>` | New array with capacity = `size()` |
| `list<T>` | `toSet()` | `set<T>` | New hash set; deduplicates |
| `set<T>` | `toList()` | `list<T>` | New list in iteration order |
| `set<T>` | `toArray()` | `array<T>` | New array in iteration order |
| `set<T>` | `asMapKeys(default: V)` | `map<T, V>` | Each element becomes a key; all values set to `default` |
| `map<K,V>` | `keys()` | `set<K>` | Live key set view |
| `map<K,V>` | `values()` | `list<V>` | Values in iteration order |
| `map<K,V>` | `entries()` | `list<(K,V)>` | Key-value pairs in iteration order |

### 9.2 Examples

**`array` ↔ `list`**

```bestie
val arr : array<int>[] = {1, 2, 3}
val ls  : list<int>    = arr.toList()   // new list from array

val ls2 : list<int> = {4, 5, 6}
val arr2 : array<int> = ls2.toArray()  // fixed array, capacity = 3
```

**`list` / `array` → `set`**

```bestie
val ls : list<int> = {1, 2, 2, 3}
val xs : set<int>  = ls.toSet()         // {1, 2, 3} — duplicates removed
```

**`set` → map keys**

```bestie
val permissions : set<str> = {"read", "write", "admin"}
val access : map<str, bool> = permissions.asMapKeys(false)
// {"read": false, "write": false, "admin": false}

access.put("read", true)
```

**`map` → key set and value list**

```bestie
val scores : map<str, int> = {"alice": 92, "bob": 85}

val names  : set<str>  = scores.keys()     // {"alice", "bob"}
val values : list<int> = scores.values()   // [92, 85]
```

### 9.3 Rules

* All `to*()` methods produce a **new, independent collection** — modifying the result does not affect the source
* `asMapKeys()` is the only conversion that takes a parameter (the default value for every key)
* `toSet()` on a source with duplicates silently deduplicates — the resulting set contains only unique elements
* `toArray()` on a `list<T>` produces an array with `capacity == size()` — the array is full immediately; `add()` would panic unless the caller uses a larger array instead
* `keys()`, `values()`, and `entries()` on a `map` produce views that reflect the current state of the map at the time of the call

---

## 10. Summary

Bestie collections are:

* Generic and type-safe
* Explicitly constructed
* Deterministic in memory layout whenever the selected representation permits it
* Ownership-aware
* Compile-time validated
* Free of hidden allocation behavior
* Explicitly immutable via `immutable` or `const`
* Universally iterable — every collection and `array<T>` work with `for/in` without conversion
* Uniformly indexed — `[i]`, `[-i]`, and `[lo..hi]` follow the same convention as core `array<T>`/`slice<T>`, exposed only where the representation supports it without hidden cost (§8a)

`list<T>` is the go-to dynamic collection. Its array-backed default is efficient and familiar. Switch to `list<T>.linked` when the access pattern calls for it.

`array<T>` (in `core/types.md`) is the fixed-capacity alternative — same indexing syntax, no dynamic growth, panics on overflow.

Conversions between any two collection types are explicit `to*()`/`as*()` method calls — no implicit coercion, no surprise allocation.

> Collections in Bestie are **explicit tools**, not abstractions with surprises — matching the language's core philosophy of predictability, safety, and control.

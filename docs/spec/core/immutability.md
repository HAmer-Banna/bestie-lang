# Bestie — Immutability Reference

This document is the **canonical reference** for immutability across all Bestie types.

It answers, for every type: what is its natural mutability? What does `val` prevent? When is `immutable` needed? Can it be `const`?

---

## 1. The Two Axes

Bestie separates two independent concerns:

**Binding immutability** — can this name be rebound to a different value?

* `val` → the binding cannot be rebound
* `var` → the binding can be rebound

**Value immutability** — can the contents of the value itself be changed?

* Depends on the type, and on `const` / `immutable`

These are independent:

```bestie
val arr : array<int>[3]    // binding frozen, elements still mutable
var s   : str = "hello"    // binding rebindable, but str has no mutation API
```

`val` alone is **not** always sufficient to prevent all mutation. The type decides what else is needed.

---

## 2. The Three Mechanisms

| Mechanism | Means | Positions | Depth |
| --------- | ----- | --------- | ----- |
| `const` | The value is fixed | `const NAME: T = <compile-time expr>` · `ptr<const T>` | Shallow — this access path |
| `immutable` | The mutation API is unavailable | `immutable class C { }` · `list<T>.immutable` | Deep |
| `val` / `var` | The binding is fixed, or not | binding · field | Binding only |

There is no fourth mechanism, and there is no per-binding freeze annotation.

### 2.1 The `immutable` rule

> **`immutable` means the mutation API is unavailable.**

A type that has **no** mutation API — `str`, `int`, `tuple`, `range<T>`, `data class`, `enum` — is already immutable, and `immutable` on it is a no-op. A type that **has** one — `list`, `set`, `map`, `array<T>`, `class` — loses it: every mutating method and every `x[i] = v` becomes a compile-time error.

That single rule replaces a table of special cases. In particular it explains why `str` needs no modifier: `str` has no mutation API at all. `s + " world"` is a **constructor**, exactly like `a + b` on `int` — not a mutation being redirected. There is nothing to remove, so there is nothing to say.

**`immutable` is freeze, never persist.** It never turns a mutating call into one that returns a new value:

```bestie
val ls : list<int>.immutable = ...
val ls2 = ls.add(4)      // ❌ compile-time error: 'add' is not available on an immutable list
```

A persistent collection — where `add` returns a new collection — is deliberately absent. `list<T>` is a `class` with an owned heap buffer, so `ls.add(4)` returning a new list would hand the caller an `own` obligation from a call that reads like an update, and a loop over it orphans one allocation per iteration. Making that cheap requires structural sharing, which requires refcounting to know when a shared node dies — and `own` guarantees exactly one owner, so there is nothing to share. Bestie has no garbage collector to absorb the difference, so the honest answer is to remove the operation rather than redirect it.

### 2.2 Why `const` is separate

`const` and `immutable` answer different questions, at different depths:

* **`const`** — *can a store instruction target this?* One access path, shallow. This is the axis that maps onto C's pointer-const forms (`memory.md` §8.4.1), which is why the spelling is kept.
* **`immutable`** — *do these methods exist?* A property of the type, transitive through everything reachable.

`const` is also the original case it was introduced for:

```bestie
const PI: float64 = 3.14159265358979323846
```

A name whose value is fixed at compile time, placed in `.rodata`, and inlined as an immediate at every use site. Assigning to it is a compile-time error. The `.rodata` placement is a *consequence* of the initializer being compile-time evaluable (`lang.md` §4.1), not a second meaning.

### 2.3 `immutable` is a weak keyword

`immutable` is **contextual**: it has meaning only immediately before `class`, and as a collection variation. It is not reserved, so `val immutable = 1` remains legal — the same treatment `data`, `value`, `virtual`, and `override` receive (`lang.md` §3.1.4).

---

## 3. Naturally Immutable Types

These types have **no mutation API**. All operations produce new values. `val` is sufficient to fully protect them, and `immutable` is a no-op.

### 3.1 Primitives

`int`, `int8`–`int64`, `uint`, `uint8`–`uint64`, `byte`, `float32`, `float64`, `bool`, `char`

* Arithmetic and logical operators always return new values
* `val n: int = 5` — `n` cannot be rebound; the integer `5` cannot be altered
* `var n: int = 5; n = 6` — valid rebinding, not mutation of the integer `5`

### 3.2 `str`

* All operations (`+`, and the `bestie.lib.strings` extensions) return a **new `str`**
* `val s: str` — `s` cannot be rebound, and the string value cannot be changed
* `immutable` does not apply — there is no mutation API to remove

### 3.3 `tuple`

Positional aggregate with no mutation API. `val t: (int, str)` freezes both the binding and the fields.

### 3.4 `range<T>`

Value type with no mutation methods. `val r = 0..10` is fully immutable.

### 3.5 `data class`

* All fields are `val` by definition — the compiler enforces this
* No setters, no mutation API anywhere in the type
* "Changing" a `data class` value means constructing a new instance
* Fields holding collections must use an `immutable` variation (`oop.md` §3.1)

```bestie
data class User {
    id: int
    name: str
}

val u = User.new(id: 1, name: "alice")
// u.name = "bob"  — ❌ no setter exists, field is val
```

### 3.6 `enum`

Closed set of values, no mutable state. Tag-only enums lower to integer constants and may be used as `const`:

```bestie
const WEEKEND_START : WeekendDays = WeekendDays.FRIDAY
```

---

## 4. `value class` — `val` Freezes the Whole Value

Because `value class` is copy-by-value (no identity, laid out inline, no heap indirection), immutability follows the primitive model:

* `val p: Point` — the **entire value is frozen**, including fields declared `var` in the class body
* `var p: Point` — the value can be replaced, and `var` fields can be mutated in place

```bestie
value class Point {
    var x: int
    var y: int
}

val p = Point.new(1, 2)
p.x = 5              // ❌ 'p' is a val binding — entire value frozen
p = Point.new(3, 4)  // ❌ binding is val

var q = Point.new(1, 2)
q.x = 5              // ✅ var binding allows field mutation
```

This is consistent with primitives: `val n: int` freezes the number entirely. A `value class` is a named bundle of values, so the same rule applies.

---

## 5. `array<T>`

`array<T>` is a core built-in with value-class semantics at the dispatch level, but at the binding level it behaves as a **fat pointer** (pointer + size + capacity). `val` prevents rebinding that fat pointer; it does **not** prevent element mutation.

| Declaration | Rebind? | Mutate elements? | Notes |
| ----------- | ------- | ---------------- | ----- |
| `var arr : array<int>[5]` | ✅ | ✅ | fully mutable |
| `val arr : array<int>[] = {1,2,3}` | ❌ | ✅ | binding frozen, elements free |
| `val arr : array<int>.immutable` | ❌ | ❌ | mutation API unavailable |
| `const arr : array<int>[] = {1,2,3}` | ❌ | ❌ | literal only — stored in `.rodata` |

`array<T>` has no builder chain, so `array<T>.immutable` is not constructed directly. It is produced by **`freeze()`**, an ownership-consuming conversion:

```bestie
val own arr = array<int>[3]
arr.add(1)
arr.add(2)

val own frozen = move arr.freeze()   // array<int>.immutable — arr is now invalid
frozen[0] = 9                        // ❌ mutation API unavailable
```

Because `move` invalidates the source, no mutable path to that storage survives. That is what makes the result safe to share across threads with no lock — see §7 and `oop.md` §14.

---

## 6. `class`, `open class`, `abstract class` — Reference Types

These are reference types. The binding holds a reference to the heap object.

* `val c: MyClass` — the reference cannot be rebound; the object **can** still be mutated through it
* `var c: MyClass` — both the reference and the object can change

Deep immutability is a property of the **type**:

```bestie
immutable class Config {
    port: int
    host: str
}

val cfg = Config.new(port: 8080, host: "localhost")
cfg.port = 9090    // ❌ compile-time error — the type is immutable
```

Every instance of an `immutable class` is deeply immutable everywhere it appears.

There is no per-binding form. A binding-level freeze would leave other references to the same object free to mutate it, so it could never carry a real guarantee — least of all the thread-safety guarantee in `oop.md` §14. If an instance must be frozen, its type is the place to say so.

An `immutable class` may not declare `var` fields or `var` properties, and may not hold a mutable collection variation. Its fields must themselves be immutable, transitively — that is what makes the guarantee deep.

---

## 7. Std-Lib Collections

`list<T>`, `set<T>`, `map<K,V>`, `deque<T>`, and `heap<T>` are reference-type `class` instances. Immutability is selected as a **variation** (`std-lib/collections.md`):

| Declaration | Rebind? | Mutate? |
| ----------- | ------- | ------- |
| `var ls : list<int>` | ✅ | ✅ |
| `val ls : list<int>` | ❌ | ✅ — the binding is frozen, the list is not |
| `val ls : list<int>.immutable` | ❌ | ❌ — mutation API unavailable |
| `const ls : set<int> = {1,2,3}` | ❌ | ❌ — literal only, `.rodata` |

An immutable collection is obtained either by constructing one directly or by freezing a mutable one:

```bestie
val own ls = list<int>.new()
ls.add(1)
ls.add(2)

val own frozen = move ls.freeze()    // list<int>.immutable — ls is now invalid
frozen.add(3)                        // ❌ 'add' is not available
```

`list<int>` and `list<int>.immutable` are **distinct types with no conversion between them**; `freeze()` is the only bridge, and it consumes its input. See `std-lib/collections.md` for the full variation rules.

---

## 8. `ptr<T>` — Two Axes, Not an Exception

`ptr<T>` uses the same two axes as the rest of Bestie. They map onto C's four pointer-const forms (`memory.md` §8.4.1):

* **Binding** (`var` / `val` / `const`) — can this name be rebound to another address? (C `int * const`)
* **Pointee** (`ptr<T>` / `ptr<const T>`) — can you write through the pointer? (C `const int *`)

| Declaration | Rebind `p`? | Write `p.val`? |
| ----------- | ----------- | -------------- |
| `var p: ptr<T>` | yes | yes |
| `val p: ptr<T>` | no | yes |
| `var p: ptr<const T>` | yes | no |
| `val p: ptr<const T>` | no | no |

* `const` here is **shallow** — it freezes exactly one level of indirection, per C. Nested pointers apply it independently at each level (`memory.md` §8.6).
* The address word is `p.addr` (`uint`); `p.toStr()` / `"${p}"` print that address, not `p.val` (`memory.md` §8.4.2)
* Index writes (`p[0] = v`) follow the same pointee rule as `p.val = v`

---

## 9. Optional and Error-Union Wrappers (`T ?` / `T ! E`)

* The **container variant** (present/absent, ok/err) is always immutable — it cannot be altered once set
* The **contained value's** mutability follows `T` (and `E`) by the rules in this document

---

## 10. Summary Table

| Type | Kind | Mutation API? | `val` enough? | `immutable` applies? | `const` valid? |
| ---- | ---- | ------------- | ------------- | -------------------- | -------------- |
| `int`, `uint*`, `float*`, `bool`, `char`, `byte` | primitive | ❌ none | ✅ | no-op | ✅ |
| `str` | value type | ❌ none | ✅ | no-op | ✅ |
| `tuple` | value type | ❌ none | ✅ | no-op | ✅ |
| `range<T>` | value type | ❌ none | ✅ | no-op | ✅ |
| `data class` | data class | ❌ none | ✅ | no-op | ❌ |
| `enum` | enum | ❌ none | ✅ | no-op | ✅ (tag-only) |
| `value class` | value class | fields only | ✅ freezes all fields | no-op | ❌ |
| `array<T>` | core built-in | ✅ | ❌ binding only | ✅ via `freeze()` | ✅ literal only |
| `slice<T>` | core view | ❌ (`slice<var T>` ✅) | ✅ | n/a — use `slice<T>` | ❌ |
| `class` / `open` / `abstract` | class | ✅ | ❌ reference only | ✅ `immutable class` | ❌ |
| `list<T>` `set<T>` `map<K,V>` | std-lib class | ✅ | ❌ binding only | ✅ `.immutable` variation | ✅ literal only |
| `deque<T>` `heap<T>` | std-lib class | ✅ | ❌ binding only | ✅ `.immutable` variation | ❌ |
| `ptr<T>` | raw pointer | ✅ | ❌ no rebind only | n/a — use `ptr<const T>` | ❌ |
| `ptr<const T>` | pointer-to-const | ❌ through it | binding: depends | n/a | ❌ |
| `T ?` / `T ! E` | core syntax | variant: ❌ / contents: depends | depends on `T` | follows `T` | ❌ |

---

## 11. Design Rationale

**One question, one word.** `const` says a store cannot target this access path. `immutable` says these methods do not exist. `val` says this name cannot be rebound. Nothing else is needed, and nothing overlaps.

**Immutability is a property of types, not of bindings.** A binding-level freeze cannot stop another reference from mutating the same object, so it can never support the thread-safety guarantee that is the main reason to want deep immutability at all. Putting it on the type makes the guarantee real.

**Freeze, never persist.** Removing an operation is honest and free. Redirecting it to allocate is neither — not in a language with explicit ownership and no garbage collector.

**Value types freeze completely; reference types declare it.** For `value class`, `data class`, `enum`, and primitives, `val` is enough because the binding *is* the value. For `class`, `array<T>`, and collections, the binding only locks the reference — the type must say the rest.

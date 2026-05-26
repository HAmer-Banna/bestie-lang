# Bestie — Immutability Reference

This document is the **canonical reference** for immutability across all Bestie types.

It answers, for every type: what is its natural mutability? What does `val` prevent? When is `@immutable val` needed? Does `.immutable` apply? Can it be `const`?

---

## 1. The Two Dimensions of Immutability

Bestie separates two independent concerns:

**Binding immutability** — can this variable be reassigned to a different value?
* `val` → the binding cannot be reassigned
* `var` → the binding can be reassigned

**Value immutability** — can the internal contents, fields, or elements of the value itself be changed?
* Depends on the type and any applied modifier

These are independent axes:

```bestie
val arr : array<int>[3]    // binding frozen, elements still mutable
var s   : str = "hello"    // binding rebindable, but str values are naturally immutable
```

`val` alone is **not** always sufficient to prevent all mutation. The type determines what else is needed.

---

## 2. The Four Immutability Mechanisms

Listed from most to least restrictive:

| Mechanism | What it prevents | Applies to |
| --------- | ---------------- | ---------- |
| `const` | Everything — compile-time only, stored in `.rodata` | Literals of primitives, `str`, `array<T>`, `set`, `map` |
| `@immutable val` | All mutation and transformation on the binding — compile-time error | All types |
| `.immutable` modifier | In-place mutation — mutations return a new object instead | std-lib collections only |
| `val` | Rebinding only — value mutability depends on the type | All bindings |

---

## 3. Naturally Immutable Types

These types have **no mutation API**. All operations produce new values; nothing modifies the original. `val` is sufficient to fully protect them in practice.

### 3.1 Primitives

`int`, `int8`, `int16`, `int32`, `int64`, `uint`, `uint8`, `uint16`, `uint32`, `uint64`, `byte`, `float32`, `float64`, `bool`, `char`

* No mutation methods exist — arithmetic and logical operators always return new values
* `val n: int = 5` — `n` cannot be reassigned; the integer `5` cannot be altered
* `var n: int = 5; n = 6` — valid rebinding, not mutation of the integer `5`
* `@immutable val` is technically allowed but redundant — there is nothing extra to prevent

### 3.2 `str`

* All operations (`+`, case, trim, etc.) return a **new `str`** — the original is always unchanged
* This is the same model as `list<T>.immutable` for collections: operations are legal but produce new values
* `val s: str` — `s` cannot be reassigned; the string value cannot be changed
* `@immutable val s: str` — equivalent to `val s: str` in practice since no mutation API exists; allowed as explicit documentation

### 3.3 `tuple`

* Positional aggregate with no mutation API
* `val t: (int, str)` — neither the binding nor the fields can be changed
* `@immutable val` is redundant

### 3.4 `range<T>`

* Value type with no mutation methods
* `val r = 0..10` — fully immutable; `@immutable val` redundant

### 3.5 `data class`

* All fields are `val` by definition — the compiler enforces this
* No setters, no mutation API anywhere in the type
* "Changing" a `data class` value means constructing a new instance
* `val u: User` — immutable from every angle
* `@immutable val` is redundant

```bestie
data class User {
    id: int
    name: str
}

val u = User.new(id: 1, name: "alice")
// u.name = "bob"  — ❌ not possible — no setter exists, field is val
```

### 3.6 `enum`

* Closed set of values; no mutable state
* Tag-only enums lower to integer constants
* `val day: WeekendDays` — immutable; `@immutable val` is redundant
* Tag-only enums may be used as `const`

```bestie
const WEEKEND_START : WeekendDays = WeekendDays.FRIDAY
```

---

## 4. `value class` — `val` Freezes the Whole Value

Because `value class` is **copy-by-value** (no identity, laid out inline, no heap indirection), immutability follows the same model as primitives:

* `val p: Point` — the **entire value is frozen**, including all fields regardless of whether they are `val` or `var` in the class definition
* `var p: Point` — the value can be replaced, and `var` fields can be mutated in place
* `@immutable val` is redundant (same as `val`)

```bestie
value class Point {
    var x: int
    var y: int
}

val p = Point.new(1, 2)
p.x = 5              // ❌ compile-time error: 'p' is a val binding — entire value frozen
p = Point.new(3, 4)  // ❌ compile-time error: binding is val

var q = Point.new(1, 2)
q.x = 5              // ✅ var binding allows field mutation
```

This is consistent with how primitives work: `val n: int` freezes the number entirely. A `value class` is just a named bundle of values — the same rule applies.

---

## 5. `array<T>` — `@immutable val` Required for Deep Freeze

`array<T>` is a core built-in with value-class semantics at the dispatch level, but at the binding level it behaves as a **fat pointer** (pointer + size + capacity). `val` prevents rebinding the fat pointer; it does **not** prevent element mutation.

| Declaration | Rebind? | Mutate elements? | Notes |
| ----------- | ------- | ---------------- | ----- |
| `var arr : array<int>[5]` | ✅ | ✅ | fully mutable |
| `val arr : array<int>[] = {1,2,3}` | ❌ | ✅ | binding frozen, elements free |
| `@immutable val arr : array<int>[] = {1,2,3}` | ❌ | ❌ | compile-time error on any mutation |
| `const arr : array<int>[] = {1,2,3}` | ❌ | ❌ | literal only — stored in `.rodata` |

`@immutable val arr` is the correct form for a runtime-created array that must never be changed.
It is also the form that makes an array safely shareable across threads without any lock.

`array<T>` does **not** support the `.immutable` builder modifier — it has no builder chain.

---

## 6. `class`, `open class`, `abstract class` — Reference Types

These are **reference types**. The binding holds a reference to the heap object.

* `val c: MyClass` — the reference cannot be rebound; the object itself **can** still be mutated through it
* `var c: MyClass` — both the reference and the object can change
* `@immutable val c: MyClass` — the reference cannot be rebound AND all field mutations through it are compile-time errors

Two forms of deep immutability are available for reference types:

**Per-binding: `@immutable val`**

Freezes one specific binding. Other references to the same object are unaffected.

```bestie
@immutable val config : ServerConfig = ServerConfig.new(port: 8080)
config.port = 9090     // ❌ compile-time error
config = other         // ❌ compile-time error
```

**Per-type: `@immutable class`**

Every instance is always deeply immutable — no `@immutable val` needed at every use site.

```bestie
@immutable class Config {
    port: int
    host: str
}

val cfg = Config.new(port: 8080, host: "localhost")
cfg.port = 9090    // ❌ compile-time error — the type itself is @immutable
```

Use `@immutable class` when the type is **conceptually** immutable (it should never have mutable instances).
Use `@immutable val` when a specific instance should be frozen at a specific binding site.

---

## 7. Std-Lib Collections — Three Levels

`list<T>`, `set<T>`, `map<K,V>`, `deque<T>`, `heap<T>` are reference-type `class` instances. Three distinct immutability levels are available:

| Declaration | Rebind? | Call mutation method? | Behaviour |
| ----------- | ------- | --------------------- | --------- |
| `var ls : list<T>` | ✅ | ✅ modifies in place | fully mutable |
| `val ls : list<T>` | ❌ | ✅ modifies in place | binding frozen |
| `val ls : list<T>.immutable` | ❌ | ✅ returns new collection | functional immutability |
| `@immutable val ls : list<T>` | ❌ | ❌ compile-time error | fully frozen |
| `const ls : set<int> = {1,2,3}` | ❌ | ❌ stored in `.rodata` | literal only |

The distinction between `.immutable` and `@immutable val`:

* **`.immutable`** — the collection is in a functional immutable state. Calling `add()` is *legal* and returns a new collection; the original is unchanged. This is the same model `str` uses. Use this for persistent / functional data patterns.
* **`@immutable val`** — calling any mutation method, even one that would return a new collection, is a **compile-time error**. Use this when mutation is a programming mistake that should never compile.

```bestie
// .immutable — functional model
val ls = list<int>.immutable.of(1, 2, 3)
val ls2 = ls.add(4)     // ✅ ls unchanged; ls2 is {1,2,3,4}

// @immutable val — hard freeze
@immutable val ls3 : list<int> = {1, 2, 3}
val ls4 = ls3.add(4)    // ❌ compile-time error: mutation on @immutable binding
```

See `std-lib/collections.md` §4.4 for full examples.

---

## 8. `ptr<T>` — The Exception

`ptr<T>` lives in the `@trusted` unsafe zone and does not follow the normal immutability model.

* `val p: ptr<T>` — the pointer cannot be rebound, but writing through it (`p[0] = v`) is still permitted
* `@immutable val p: ptr<T>` — prevents rebinding, but does **not** prevent writes through the pointer. Pointer writes bypass the language-level immutability model
* There is no `const T*` analogue in the surface language — pointer target immutability is managed through the `@trusted` discipline in `memory.md`

---

## 9. Container Types: `option<T>` and `result<T, E>`

These are wrapper types. Their immutability follows two layers:

* The **container variant** (`Present`/`None`, `Ok`/`Err`) is always immutable — it cannot be altered once set
* The **contained value's** mutability follows `T` (and `E`) according to the rules in this document

```bestie
val o : option<list<int>> = option.Present({1, 2, 3})
// o itself cannot be rebound
// the list inside can still be mutated

@immutable val o2 : option<list<int>> = option.Present({1, 2, 3})
// o2 frozen; the list inside is also frozen
```

---

## 10. Summary Table

| Type | Kind | Naturally immutable? | `val` sufficient for full immutability? | `.immutable` builder? | `@immutable val` useful? | `const` valid? |
| ---- | ---- | -------------------- | --------------------------------------- | --------------------- | ------------------------ | -------------- |
| `int`, `uint*`, `float*`, `bool`, `char`, `byte` | primitive | ✅ | ✅ | ❌ | redundant | ✅ |
| `str` | value type | ✅ | ✅ | ❌ | redundant | ✅ |
| `tuple` | value type | ✅ | ✅ | ❌ | redundant | ✅ |
| `range<T>` | value type | ✅ | ✅ | ❌ | redundant | ✅ |
| `data class` | data class | ✅ (all val fields) | ✅ | ❌ | redundant | ❌ |
| `enum` | enum | ✅ (no state) | ✅ | ❌ | redundant | ✅ (tag-only) |
| `value class` | value class | ✅ when val-bound | ✅ (freezes all fields) | ❌ | redundant | ❌ |
| `array<T>` | core built-in | ❌ (elements mutable) | ❌ (binding only) | ❌ no builder | ✅ needed | ✅ literal only |
| `class` | class | ❌ | ❌ (reference only) | ❌ | ✅ needed | ❌ |
| `open class` | class variant | ❌ | ❌ (reference only) | ❌ | ✅ needed | ❌ |
| `abstract class` | class variant | ❌ | ❌ (reference only) | ❌ | ✅ needed | ❌ |
| `list<T>` | std-lib class | ❌ | ❌ (binding only) | ✅ new-object model | ✅ hard freeze | ✅ literal only |
| `set<T>` | std-lib class | ❌ | ❌ (binding only) | ✅ new-object model | ✅ hard freeze | ✅ literal only |
| `map<K,V>` | std-lib class | ❌ | ❌ (binding only) | ✅ new-object model | ✅ hard freeze | ✅ literal only |
| `deque<T>` | std-lib class | ❌ | ❌ (binding only) | ✅ new-object model | ✅ hard freeze | ❌ |
| `heap<T>` | std-lib class | ❌ | ❌ (binding only) | ✅ new-object model | ✅ hard freeze | ❌ |
| `ptr<T>` | unsafe built-in | ❌ | ❌ | ❌ | ⚠️ binding only | ❌ |
| `option<T>` | wrapper (enum-like) | variant: ✅ / contents: depends | depends on T | ❌ | ✅ freezes both layers | ❌ |
| `result<T,E>` | wrapper (enum-like) | variant: ✅ / contents: depends | depends on T, E | ❌ | ✅ freezes both layers | ❌ |

---

## 11. Design Rationale

Bestie's immutability model is built on three principles:

**Explicit over implicit** — no type silently becomes immutable or mutable based on context. Every constraint is visible at the declaration site.

**Layered, not monolithic** — `val`, `.immutable`, and `@immutable` address different problems. `val` prevents accidents at the binding level. `.immutable` enables functional data patterns. `@immutable` is a hard compiler guarantee. Choosing the right level is intentional.

**Value types freeze completely; reference types require an explicit freeze** — for `value class`, `data class`, `enum`, and primitives, `val` is enough because the binding IS the value. For reference types (`class`, `array<T>`, collections), `val` only locks the reference — the target object can still be mutated unless `@immutable val` is applied.

This design means the programmer always knows, from the declaration alone, exactly what can and cannot change.

# Bestie Language — Core Specification

## 1. Overview

Bestie is a native, compiled programming language designed from first principles for **systems programming and backend engineering**.

Bestie is not built around a single paradigm.
Object-oriented programming, functional programming, and low-level control are treated as **explicit tools**, not ideologies.

Bestie prioritizes:

* Compile-time resolution
* Deterministic execution
* Predictable memory layout
* Zero-cost abstractions
* Long-term stability

---

## 2. Core Design Principles

The Bestie compiler resolves **everything that is resolvable** at compile time, including:

* Type inference
* Generic specialization (monomorphization)
* Memory layout, padding, and alignment
* Protocol dispatch
* Inline expansion
* Lifetime and destructor placement
* Ownership validation
* Pointer correctness
* Builder chains
* Error handling paths
* Loop lowering and unrolling opportunities
* Devirtualization
* Concurrency safety for ownership/sharing rules (`own/ref`)

The runtime is intentionally minimal.

**Golden Rule**

> If something can be resolved at compile time, it must be.

---

## 3. Language Structure

A Bestie program consists of:

* Packages
* Modules
* Types
* Functions
* Protocols
* Annotations

The core language is **small and sealed**.

Higher-level abstractions live in:

* `std-lib`
* `std-api`
* `std-framework`

The core defines **semantic guarantees**.
Higher layers provide **convenience and extensibility**.

---

## 4. Variables and Bindings

Bestie provides three binding forms.

### 4.1 `const` — Compile-Time Constant

`const` defines an immutable binding and immutable value resolved entirely at compile time.

Properties:

* Immutable binding and value
* No runtime allocation
* No mutation through references
* Cannot reference mutable or runtime memory
* Stored in read-only memory

Valid scopes:

* File
* Class
* Protocol

Invalid scopes:

* Function parameters
* Runtime locals

Intended use:

* Mathematical constants
* Compile-time configuration
* Type-level invariants

---

### 4.2 `val` — Immutable Binding (Default)

`val` defines an immutable binding to a value.

The **binding is immutable**.
The **value mutability depends on its type**.

Properties:

* Runtime values allowed
* Preferred default
* Encourages value-oriented programming

File-level `val` must be annotated with `@immutable`.

---

### 4.3 `var` — Mutable Binding (Restricted)

`var` defines a mutable binding and is deliberately restricted.

Allowed only:

* Local variables
* Class properties with explicit accessors

Disallowed:

* File scope
* Protocols
* `data`, `value`, and `enum` declarations

---

### 4.4 Multiple Value Declarations

Multiple value declarations are **binding syntax sugar**, not data structures.

```bestie
val x, y, z = 5, 6, 3
```

Rules:

* RHS values must share the **same type**
* No tuple is created
* Binding is positional
* Lowered to independent bindings at compile time
* Zero runtime cost

---

### 4.5 Tuples and Destructuring

Tuples are **first-class heterogeneous value types**.

```bestie
val (x, y, z) = (5, 6, 'c')
```

Properties:

* Heterogeneous allowed
* Compile-time layout known
* No implicit allocation
* Explicit tuple semantics via parentheses

---

### 4.6 Ignoring Values with `_`

`_` explicitly discards values without creating bindings.

Properties:

* No lifetime
* No storage
* Compile-time only
* Suppresses unused diagnostics

---

## 5. Types

### 5.1 Primitive Types

Primitive value types map directly to machine types:

| Type                              | Width         | Notes                                    |
| --------------------------------- | ------------- | ---------------------------------------- |
| `int8`, `int16`, `int32`, `int64` | Fixed         | Signed integers, explicit width          |
| `uint8`, `uint16`, `uint32`, `uint64` | Fixed     | Unsigned integers, explicit width        |
| `byte`                            | 8-bit         | Alias for `uint8`. Used for raw memory.  |
| `int`                             | Pointer-sized | Signed. Equivalent to C `intptr_t`. Use for indices, sizes, offsets. |
| `uint`                            | Pointer-sized | Unsigned. Equivalent to C `uintptr_t` / `size_t`. |
| `float32`, `float64`              | 32 / 64-bit   | IEEE 754                                 |
| `bool`                            | 1-bit logical | `true` / `false`                         |
| `char`                            | 32-bit        | Unicode scalar value (valid codepoint)   |

`double` does not exist. Use `float64`.

**`int` and `uint` — pointer-sized by design:**

`int` and `uint` are not ambiguous — their width is the pointer width of the target platform. They exist specifically for indices, sizes, counts, and pointer arithmetic. Using `int32` or `int64` is correct when the bit width matters independently of the platform.

Mixing `int` with fixed-width types (`int32`, `int64`, etc.) requires an explicit cast. No silent narrowing or widening.

**`byte` and `uint8`:**

`byte` is an alias for `uint8`. They are the same type — no cast required between them. `byte` is the idiomatic name when working with raw memory, buffers, and binary data. `uint8` is idiomatic when the numeric value matters.

All primitives:

* No headers
* Stack-friendly
* No deallocation
* Deterministic layout

---

### 5.2 Core Value Types

Includes:

* `str` (immutable, UTF-8)
* `tuple`
* `ptr<T>`

**`str` and `char` indexing:**

`str` is UTF-8 encoded. Two indexing modes exist to separate speed from correctness:

| Operation        | Returns | Meaning                                      |
| ---------------- | ------- | -------------------------------------------- |
| `s[i]`           | `byte`  | Raw UTF-8 byte at byte offset `i`. C-speed.  |
| `s.char(i)`      | `char`  | Unicode scalar at codepoint index `i`.       |
| `s.chars()`      | iterator over `char` | Full Unicode iteration.         |

`s[i]` is direct memory access — fast, no validation. Use it when you are working with bytes.
`s.char(i)` is the human-readable path — validates and decodes. Use it when you care about characters.

No hidden cost. The fast path stays fast. The correct path is explicit.

Properties:

* Binding immutability defaults to `val`/`const`
* Value mutability depends on type qualifiers
* Efficient copy semantics
* No hidden allocation

---

### 5.3 Core Collections

Bestie includes exactly one collection in the core language: `list<T>`.

`list<T>` is built-in and available without `import`.
It participates directly in the language's type, memory, and loop rules.

Core `list<T>` supports:

* `list<T>.build()` with no import
* Built-in variations such as `array` and `linked`
* Indexing via `xs[i]`
* List literals such as `{1,2,3}` when the target type is `list<T>`
* Sized array forms such as `list<int>[10]` and `list<int>[2][3]`
* Core methods such as `add`, `remove`, `get`, `insert`, and `indexOf`
* Ownership, immutability, and concurrency semantics defined by the core specification
* Builder-chain resolution at compile time

Bestie does **not** provide a separate built-in array type.
Array semantics are expressed through the core `list<T>`.
The default `list<T>` is array-backed unless another core variation is selected.

All other collections live in `bestie.lib.collections`, including:

* `set<T>`
* `map<K,V>`
* `deque<T>`
* `heap<T>`

Example:

```bestie
val xs = list<int>.linked().build()
```

---

## 6. Casting and Type Rules

Bestie requires **explicit casting**.

No implicit numeric promotion or narrowing is allowed.

---

### 6.1 Type Aliases — Newtypes (`type X as Y`)

`type X as Y` declares a **newtype**: a distinct type with the same underlying representation as `Y`.

```bestie
type Meters as float64
type Seconds as float64
```

**Properties:**

* `X` is a distinct type from `Y` — they cannot be used interchangeably
* No implicit conversion in either direction
* Zero runtime overhead — same memory layout as `Y`
* `X` automatically inherits all methods and operators of `Y`
* `X` may implement additional protocols

**Casting:**

```bestie
val m: Meters  = 10.0 as Meters    // wrap
val raw: float64 = m as float64    // unwrap
```

**Type safety:**

```bestie
val m: Meters  = 10.0 as Meters
val s: Seconds = 5.0 as Seconds

val bad = m + s    // ❌ compile error: Meters and Seconds are distinct types
val good = m + m   // ✅ Meters inherits float64's + operator
```

**If full interchangeability is needed**, use the original type directly. `type X as Y` is always a newtype — there is no alias-without-distinction form.

---

## 7. Generics

Generics are **compile-time only**.

Properties:

* Full monomorphization
* No runtime type information
* No type erasure
* Predictable layout and performance

---

### 7.1 Generic Constraints

Generic type parameters may be constrained using `ext` and `impl`, consistent with the rest of the type system.

**Protocol constraint:**

```bestie
fun <T impl Comparable> sort(xs: list<T>): list<T>
fun <T impl Hashable> dedupe(xs: list<T>): list<T>
```

**Class constraint:**

```bestie
fun <T ext Shape> render(t: T): void
```

**Multiple constraints of the same kind** — comma-separated, no need to repeat the keyword:

```bestie
fun <T impl Comparable, Hashable> index(t: T): int
```

**Mixed constraints** — restart with the new keyword:

```bestie
fun <T ext Animal impl Printable, Serializable> describe(t: T): str
```

**Multiple type parameters:**

```bestie
fun <K impl Hashable, V> lookup(m: map<K, V>, key: K): V ?
```

**Constraints on types and protocols:**

```bestie
data class Pair<T impl Equable> {
    first: T
    second: T
}

protocol Container<T impl Hashable> {
    fun contains(item: T): bool
}
```

Rules:

* `T ext X` — `T` must be a subclass of `X` (`X` must be `open` or `abstract`)
* `T impl P` — `T` must implement protocol `P`
* Comma continues the current `impl` or `ext` group
* A new `impl` / `ext` keyword starts a new group
* All constraints are AND semantics — every constraint must be satisfied
* Violations are **compile-time errors**
* No runtime dispatch introduced — still fully monomorphized

---

## 8. Control Flow

### `if`

`if` is both a **statement** and an **expression**.

As an expression, `if` must produce a value:

```bestie
val x: int = if (cond) 4 else 0
```

If a value does not have a natural empty representation, `Option<T>` must be used.

As a statement, `if` may omit `else`.
When used inside a function without `else`, the function becomes **partial**:

```bestie
fun f(): bool ? = if (cond) return true
```

See `fp.md` for partial functions.

Properties:

* Statement and expression
* Expressions must return values
* Missing branch → partial function (`?`)

### `switch`

`switch` is both a **statement** and an **expression**.

Properties:

* No fallthrough
* Exhaustiveness enforced when expression
* Fully compile-time analyzable

```bestie
val x = switch (v) {
  1 => 10
  2 => 20
  else => 0
}
```

---

## 9. Loops

Supports:

* `for`
* `for in`
* `while`

Loops may include an `else` clause (similar to Python) when the loop completes without `break`.

Loops may be used as **expressions when compile-time resolvable**:

```bestie
val x: int = for (i = 0; i < 5; i++) i + 5
val xs: list<int> = for (i in 0..3) i * 2
```

While loop example:

```bestie
var i = 0
while (i < 10) {
  print(i)
  i += 1
}
```

This provides comprehension-style behavior without stream abstractions.

---

## 10. Syntax Rules

* Parentheses required for control flow
* Braces required for multi-line bodies
* Formatting enforced by tool, not compiler

---

## 11. Operators

Logical, bitwise, identity, and compile-time introspection operators supported.

No hidden operator behavior.

---

## 12. Functions (Overview)

* Static dispatch by default
* No hidden allocation
* Compile-time resolvable

See `fp.md`.

---

## 13. OOP (Overview)

* Closed classes by default
* Explicit inheritance
* Protocol-driven polymorphism

See `oop.md`.

---

## 14. Memory Model (Overview)

Manual, deterministic memory model:

* `own`
* `ref`
* `ptr<T>`
* Explicit unsafe boundaries for low-level operations (`ptr`, FFI, manual free)

See `memory.md`.

---

## 15. Concurrency (Overview)

* Explicit
* Compile-time validated
* No hidden sharing

See `concurrency.md`.

---

## 16. Annotations

Compile-time only, zero runtime cost.

See `annotations.md`.

---

## 17. Error Handling

Bestie avoids hidden null-like states and hidden exception flow.

Mechanisms:

* Complete returns
* Partial returns (`?`)
* `Option<T>`
* Explicit errors

See `exceptions.md`.

---

## 18. Integer Overflow

Overflow behavior in Bestie is **defined, deterministic, and zero-cost in release builds**.

### 18.1 Unsigned Integers — Always Wrap

Unsigned overflow always wraps, unconditionally:

```bestie
val x: uint8 = 255
val y = x + 1   // y = 0, always
```

### 18.2 Signed Integers — Wrap in Release, Trap in Debug

In release builds, signed overflow wraps (two's complement). No cost, no surprise.
In debug builds, signed overflow traps at runtime to surface bugs early.

```bestie
val x: int8 = 127
val y = x + 1   // release: y = -128
                // debug:   runtime trap
```

Build mode is set in `bestie-project.toml`. No per-expression overhead in release.

### 18.3 Explicit Overflow Operators

When wrapping, saturating, or checked behavior is intentionally required, use explicit operators:

| Operator | Behavior                              | Use case                          |
| -------- | ------------------------------------- | --------------------------------- |
| `+%`     | Always wraps (unsigned semantics)     | Hash functions, ring buffers      |
| `-%`     | Always wraps                          | Modular arithmetic                |
| `*%`     | Always wraps                          | Fixed-width math                  |
| `+\|`    | Saturates at type max/min             | DSP, graphics, clamped counters   |
| `-\|`    | Saturates at type max/min             | Audio, signal processing          |
| `*\|`    | Saturates at type max/min             | Image pixel math                  |
| `+!`     | Always traps on overflow              | Safety-critical paths             |
| `-!`     | Always traps on overflow              | Checked arithmetic                |
| `*!`     | Always traps on overflow              | Checked arithmetic                |

```bestie
val a = x +% y    // wraps
val b = x +| y    // saturates
val c = x +! y    // traps
```

Rules:

* Explicit operators override build-mode behavior
* No hidden branching in release for `+%` and `+|`
* `+!` compiles to a checked instruction; cost is explicit and local

---

## 19. `defer`

`defer` schedules a statement to execute at the **end of the enclosing scope**, regardless of how the scope exits — normal return, early return, or exception unwind.

It is compile-time lowered. There is no runtime mechanism, no allocation, and no overhead.

### 19.1 Basic Usage

```bestie
fun readConfig(path: str): str ! IoError {
    val f = try file.open(path)
    defer f.close()              // runs on every exit path

    return try f.readAll()
}
```

### 19.2 Multiple Defers — LIFO Order

Multiple `defer` statements in the same scope run in **reverse declaration order**:

```bestie
val db = try db.connect()
defer db.close()          // runs second

val tx = try db.beginTx()
defer tx.rollback()       // runs first
```

### 19.3 Defer Inside Loops

`defer` inside a loop fires at the **end of each iteration**, not the end of the function:

```bestie
for (item in items) {
    val f = try file.open(item.path)
    defer f.close()       // closes after each iteration
    process(f)
}
```

### 19.4 Rules

* `defer` executes at the end of its **enclosing scope block**, not the function
* `defer` body cannot use `return`, `throw`, or `try`
* `defer` captures the variables it references **by binding** at declaration time
* Multiple defers in one scope execute **LIFO**
* Compile-time lowered — no runtime mechanism

---

## 20. Stability

* Core is sealed
* Backward compatibility is mandatory
* Experimental features require flags
* Higher layers evolve independently

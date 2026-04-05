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

* `int8`, `int16`, `int32`, `int64`
* `uint8`, `uint16`, `uint32`, `uint64`
* `int`, `uint` (width inferred by target)
* `float32`, `float64`
* `bool`
* `char`

`double` does not exist.

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

Aliases using `type X as Y` introduce **no new runtime type**.

---

## 7. Generics

Generics are **compile-time only**.

Properties:

* Full monomorphization
* No runtime type information
* No type erasure
* Predictable layout and performance

---

## 8. Control Flow

### `if`

`if` is both a **statement** and an **expression**.

As an expression, `if` must produce a value:

```bestie
val x: int = if (cond) 4 else 0
```

If a value does not have a natural empty representation, `option<T>` must be used.

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
* `option<T>`
* Explicit errors

See `exceptions.md`.

---

## 18. Stability

* Core is sealed
* Backward compatibility is mandatory
* Experimental features require flags
* Higher layers evolve independently

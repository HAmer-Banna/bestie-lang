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
* Concurrency safety (races, deadlocks, illegal sharing)

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
* `data`, `value`, and `single` classes

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
* collections (`list`, `set`, `map`, `deque`, `heap`)

Properties:

* Immutable by default
* Efficient copy semantics
* No hidden allocation

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

* Statement and expression
* Expressions must return values
* Missing branch → partial function (`?`)

### `switch`

* No fallthrough
* Exhaustiveness enforced when expression
* Fully compile-time analyzable

---

## 9. Loops

Supports:

* `for`
* `for in`
* `while`

Loops may be used as **expressions when compile-time resolvable**.

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
* `ptr`

See `memory.md`.

---

## 15. Concurrency (Overview)

* Explicit
* Compile-time validated
* No hidden sharing

See `concurrency.md`.

---

## 16. Collections (Overview)

Deterministic, ownership-aware, generic.

See `collections.md`.

---

## 17. Annotations

Compile-time only, zero runtime cost.

See `annotations.md`.

---

## 18. Error Handling

Bestie avoids null and hidden exceptions.

Mechanisms:

* Complete returns
* Partial returns (`?`)
* `Option<T>`
* Explicit errors

See `exceptions.md`.

---

## 19. Stability

* Core is sealed
* Backward compatibility is mandatory
* Experimental features require flags
* Higher layers evolve independently

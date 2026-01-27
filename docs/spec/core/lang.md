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

Bestie is conceptually:
**Zig + Kotlin**, without inheriting the weaknesses of either.

---

## 2. Core Design Principles

The Bestie compiler resolves **everything that is resolvable** at compile time, including:

* Type inference
* Generic specialization (monomorphization)
* Memory layout
* Padding and alignment
* Protocol dispatch
* Inline expansion
* Destructor placement
* Ownership validation
* Pointer safety
* Builder chains
* Error handling paths
* Loop unrolling opportunities
* Devirtualization
* Concurrency safety (races, deadlocks, sharing rules)

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

The core defines guarantees.
APIs define convenience.

---

## 4. Variables and Bindings

Bestie provides three binding forms.

### 4.1 `const` — Compile-Time Constant

`const` defines an immutable binding and immutable value resolved entirely at compile time.

Properties:

* Immutable binding and value
* No runtime allocation
* No mutation through references
* Cannot reference mutable memory

Valid scopes:

* File
* Class
* Protocol

Invalid scopes:

* Function parameters
* Runtime locals

`const` is intended for:

* Mathematical constants
* Compile-time configuration
* Type-level invariants

---

### 4.2 `val` — Immutable Binding (Default)

`val` defines an immutable binding to a value that may be mutable depending on its type.

Properties:

* Runtime values allowed
* Preferred default
* Encourages immutability

File-level `val` must be annotated with `@immutable`.

---

### 4.3 `var` — Mutable Binding (Restricted)

`var` defines a mutable binding and is deliberately restricted.

Allowed only:

* Local variables
* Class properties with explicit getters/setters

Disallowed:

* File scope
* Protocols
* Data/value/single classes

---

## 5. Types

### 5.1 Primitive Types

Primitive value types map directly to machine types:

* `byte`
* `int32`, `int64`
* `uint32`, `uint64`
* `float32`, `float64`
* `int`, `uint`, `float` (width inferred by target)
* `bool`
* `char`

All primitives:

* Have no headers
* Are stack-friendly
* Require no deallocation

---

### 5.2 Core Value Types

Value types include:

* `str` (UTF-8, immutable)
* `tuple`
* `ptr<T>`
* Immutable collection literals

Value types:

* Are immutable by default
* Are copied efficiently
* Have no hidden allocation

---

## 6. Functions (Overview)

Functions are declared using `fun`.

Properties:

* Static dispatch by default
* Explicit dynamic dispatch via annotations
* No implicit heap allocation
* No hidden captures

Full specification:
➡ `fp.md`

---

## 7. Object-Oriented Programming (Overview)

Bestie supports OOP with explicit control.

Core concepts:

* Classes are closed and inlined by default
* Explicit inheritance
* Explicit polymorphism
* Protocols for behavior

There is no implicit virtual dispatch.

Full specification:
➡ `oop.md`

---

## 8. Memory Model (Overview)

Bestie uses **manual, deterministic memory management** with compile-time safety.

Core primitives:

* `ptr<T>`
* `own<T>`
* `ref<T>`

Properties:

* Explicit `.new()` allocation
* Explicit `.free()` / `.freeDeep()`
* No GC
* No null
* No undefined behavior
* No dangling pointers (compile-time enforced)

Full specification:
➡ `memory.md`

---

## 9. Performance Guarantees

Bestie guarantees performance equivalent to hand-written C by design:

* Classes are inlined by default
* No GC
* No hidden indirection
* No implicit virtual dispatch
* Optimal memory layout (padding & alignment)
* Generic specialization
* Zero-cost error handling
* Deterministic destructors
* Bounds checks elided when provably safe
* No runtime type system

These are guarantees, not optimizations.

---

## 10. Concurrency (Overview)

Concurrency is:

* Explicit
* Compile-time validated
* Free of hidden sharing

No implicit locks or shared mutable state exist.

Full specification:
➡ `concurrency.md`

---

## 11. Annotations

Annotations:

* Are compile-time only
* Introduce no runtime overhead
* Cannot alter core semantics

Examples:

* `@inline`
* `@virtual`
* `@override`
* `@immutable`

Full specification:
➡ `annotation.md`

---

## 12. Error Handling (Overview)

Bestie does not use nulls or implicit exceptions.

Error mechanisms:

1. Complete returns
2. Partial returns (`?`)
3. `option<T>`
4. Explicit error values

Full specification:
➡ `errors.md`

---

## 13. What Is Intentionally Excluded

Bestie deliberately excludes:

* Garbage collection
* Reflection
* Macros
* Unsafe blocks
* Runtime metaprogramming
* Implicit dynamic dispatch
* Global mutable state

If a feature exists, it must:

1. Be safe by default
2. Have zero runtime cost
3. Be predictable
4. Remain valid long-term

---

## 14. Stability & Versioning

* Core language is sealed
* Backward compatibility is mandatory
* Experimental features require compiler flags
* APIs evolve without breaking core guarantees


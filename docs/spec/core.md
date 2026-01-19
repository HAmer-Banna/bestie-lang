# Bestie Language — Core Specification

## 1. Overview

Bestie is a **native, compiled, practical programming language** designed from day one to serve **both system programming and backend engineering**.

Bestie is not built around a single paradigm. Instead, it provides **object-oriented programming, functional programming, and low-level control** as *tools*, not ideologies.

Design principles:
- Fast compilation
- Explicit behavior
- No undefined behavior
- No garbage collection
- No nulls
- Best-in-class memory layout
- Small, sealed core

> Bestie aims to be: **Zig + Kotlin**, without inheriting the regrets of either.

---

## 2. Language Structure

A Bestie program is composed of:
- Packages
- Modules
- Types
- Functions
- Protocols
- Annotations
- Explicit APIs layered on top of a minimal core

The core language is **intentionally small**. Advanced abstractions live in:
- `std-lib`
- `std-api`
- `std-framework`

---

## 3. Basic Types

Bestie provides a minimal but expressive set of built-in types.

### 3.1 Primitive Value Types

All primitives are **value types**:

- `int`, `int8`, `int16`, `int32`, `int64`
- `uint`, `uint8`, `uint16`, `uint32`, `uint64`
- `float`, `float32`, `float64`
- `bool`
- `char`

These types:
- Are stack-friendly
- Have no hidden headers
- Do not require manual deallocation

---

### 3.2 Core Value Types

The following are also **value classes**:

- `str`
- `tuple`
- `ptr<T>`
- `option<T>` (enum class)
- collection literals (`list`, `set`, `map` — immutable by default)

Value classes:
- Are immutable by default
- Can be copied cheaply
- Do not require `free()` unless they own heap memory internally

---

## 4. Generics

Bestie supports **compile-time generics** with no runtime overhead.

```bestie
fun max<T>(a: T, b: T): T
Properties:
	•	Monomorphized at compile time
	•	No type erasure
	•	No runtime RTTI requirement
	•	Compatible with static polymorphism

Generics are part of the core language and do not depend on OOP or FP features.

⸻

5. Functions and Lambdas (Overview)

Functions are declared using fun:
fun add(a: int, b: int): int {
    return a + b
}

Lambda syntax:
val f = (x: int) => x * 2

Bestie supports:
	•	Overloading
	•	Static dispatch by default
	•	Explicit dynamic dispatch (@virtual)
	•	Partial functions (?)
	•	Option returns
	•	Error returns

Full details are specified in:

➡ [fp.md] — Functional Programming in Bestie

⸻

6. Object-Oriented Programming (Overview)

Bestie supports OOP as a controlled, explicit toolset.

Core concepts:
	•	Classes (closed by default)
	•	Data / value / enum classes
	•	Single classes
	•	Protocols (static and virtual)
	•	Group protocols
	•	Explicit inheritance rules
	•	Explicit polymorphism

Bestie avoids:
	•	Implicit virtual dispatch
	•	Inheritance ambiguity
	•	Fragile base classes

Full specification lives in:

➡ [oop.md] — Object-Oriented Programming in Bestie

⸻

7. Memory Model (Overview)

Bestie is manual-memory, safety-oriented, and deterministic.

Core concepts:
	•	ptr<T>
	•	own<T>
	•	ref<T>
	•	Explicit allocation via .new()
	•	Explicit deallocation via .free() / .freeDeep()
	•	No garbage collection
	•	No null values
	•	No undefined behavior

Memory APIs exist to reduce boilerplate, not to hide behavior.

Full rules and guarantees are defined in:

➡ [memory.md] — Memory Core & Memory API

⸻

8. Concurrency Model (Overview)

Concurrency in Bestie is:
	•	Explicit
	•	Compile-time validated
	•	Free of hidden synchronization
	•	Separate from memory ownership

Core primitives:
	•	OS threads
	•	Lightweight threads
	•	Explicit scheduling
	•	No implicit shared mutable state

High-level abstractions (actors, pools, etc.) live in APIs, not the core.

Full details in:

➡ [concurrency.md] — Concurrency Core & API

⸻

9. Annotations and Plugins (Overview)

Annotations in Bestie:
	•	Are resolved at compile time
	•	Do not introduce runtime overhead
	•	Do not mutate the core language

Examples:
	•	@virtual
	•	@override
	•	@immutable
	•	@inline
	•	@expose

Custom annotations require compiler plugins and cannot alter core semantics.

Full specification:

➡ [annotation.md] — Annotations & Compiler Plugins

⸻

10. Error Handling and Exceptions (Overview)

Bestie does not use:
	•	Null
	•	Nil
	•	Undefined
	•	Implicit exceptions

Error handling options:
	1.	Complete return
	2.	Partial return (?)
	3.	option<T>
	4.	Error returns (Zig-style)

Errors are explicit and enforced by the compiler.

Detailed rules are documented in:

➡ [errors.md]

⸻

11. What Is Intentionally Not in Core

The following are deliberately excluded:
	•	Garbage collection
	•	Macros
	•	Reflection
	•	Unsafe blocks
	•	Runtime metaprogramming
	•	Implicit dynamic dispatch
	•	Global mutable state

If something exists in Bestie, it must:
	1.	Be useful
	2.	Be safe by default
	3.	Not hurt performance
	4.	Never become “don’t use this” advice

⸻

12. Specification Stability
	•	The core is sealed
	•	Behavior is explicit
	•	Backward compatibility is a priority
	•	Advanced features evolve via APIs and frameworks

⸻

13. Next Documents
	•	oop.md — Classes, Protocols, Inheritance, Polymorphism
	•	fp.md — Functions, Lambdas, Partial Functions
	•	memory.md — Ownership, Pointers, Allocation
	•	concurrency.md — Threads and Scheduling
	•	annotation.md — Compile-Time Extensions
	•	errors.md — Error Model
	•	effective-bestie/ — Best Practices & Guidelines

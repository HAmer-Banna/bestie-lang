Bestie Language — Core Specification

1. Overview

Bestie is a native, compiled, practical programming language designed from first principles to serve both system programming and backend engineering.

Bestie is not centered around a single paradigm. Instead, it treats object-oriented programming, functional programming, and low-level control as explicit tools, not ideologies.

Design principles:
	•	Fast compilation
	•	Explicit behavior
	•	No undefined behavior
	•	No garbage collection
	•	No null values
	•	Predictable and optimal memory layout
	•	Small, sealed core language

Bestie aims to be: Zig + Kotlin, without inheriting the regrets of either.

⸻

2. Language Structure

A Bestie program is composed of:
	•	Packages
	•	Modules
	•	Types
	•	Functions
	•	Protocols
	•	Annotations
	•	Explicit APIs layered on top of a minimal core

The core language is intentionally small and sealed. Higher-level abstractions live in:
	•	std-lib
	•	std-api
	•	std-framework

The core defines what the language guarantees. APIs define what the ecosystem provides.

⸻

3. Variables and Bindings

Bestie provides three distinct ways to define variables, each with clearly defined guarantees and constraints.

3.1 const — Compile-Time Constants

const defines an immutable binding and immutable value that must be fully resolved at compile time.

Properties:
	•	Binding is immutable
	•	Value is immutable
	•	Must be resolved at compile time
	•	Must be initialized at declaration
	•	No runtime allocation
	•	No setters or mutation of referenced data

Valid scopes:
	•	File level
	•	Class level
	•	Protocol level

Invalid scopes:
	•	Function parameters
	•	Local variables with runtime values

Examples:

const PI: float64 = 3.141592653589793
const MAX_RETRIES: int = 5

Compile-time object construction is allowed only if fully resolvable at compile time:

const config: Config = Apple.config() // only if resolved at compile time

Restrictions:
	•	const cannot be assigned to runtime objects
	•	const cannot depend on runtime values
	•	const cannot reference mutable memory

const is intended for:
	•	Mathematical constants
	•	Compile-time configuration
	•	Protocol- and type-level invariants

⸻

3.2 val — Immutable Binding (Preferred Default)

val defines an immutable binding, while the value itself may be mutable, depending on its type.

Properties:
	•	Binding is immutable
	•	Value mutability depends on type
	•	Runtime values are allowed
	•	Most commonly used variable form

Valid scopes:
	•	Local scope
	•	Function parameters
	•	Class properties
	•	Data / value / single classes

Invalid scopes:
	•	Protocol definitions

Examples:

val x: int = 10
val user = User("Alice")

File-Level val
At file scope, val must be annotated with @immutable.

@immutable
val defaultTimeout: int = 30

Rules:
	•	@immutable enforces value immutability
	•	Without @immutable, the compiler emits a warning
	•	File-level mutable state is discouraged by design

val is the default and recommended choice for most use cases.

⸻

3.3 var — Mutable Binding (Restricted)

var defines a mutable binding and represents the weakest form of variable definition in Bestie.

Properties:
	•	Binding is mutable
	•	Value is mutable
	•	Explicitly restricted usage

Valid scopes:
	•	Local variables
	•	Class properties with explicit get / set

Invalid scopes:
	•	File level
	•	Protocols
	•	Data classes
	•	Value classes
	•	Single classes

Examples:

class Counter {
    var value: int
        get
        set
}

var is intended primarily for:
	•	Properties requiring controlled mutation
	•	State with explicit getters and setters

Unrestricted mutable state is deliberately disallowed.

⸻

4. Basic Types

Bestie provides a minimal but expressive set of built-in types.

4.1 Primitive Value Types

All primitive types are value types:
	•	int, int8, int16, int32, int64
	•	uint, uint8, uint16, uint32, uint64
	•	float, float32, float64
	•	bool
	•	char

Guarantees:
	•	Stack-friendly layout
	•	No hidden headers
	•	No implicit heap allocation
	•	No manual deallocation required

⸻

4.2 Core Value Types

The following are also value classes:
	•	str
	•	tuple
	•	ptr<T>
	•	Collection literals (list, set, map — immutable by default)

Value classes:
	•	Are immutable by default
	•	Can be copied efficiently
	•	Require no free() unless owning heap memory internally

⸻

5. Generics

Bestie supports compile-time generics with zero runtime overhead.

fun max<T>(a: T, b: T): T

Properties:
	•	Monomorphized at compile time
	•	No type erasure
	•	No runtime RTTI
	•	Compatible with static polymorphism

Generics are part of the core language, independent of OOP or FP features.

⸻

6. Functions and Lambdas (Overview)

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
	•	option<T> returns
	•	Error returns

Full specification:

➡ fp.md — Functional Programming in Bestie

⸻

7. Object-Oriented Programming (Overview)

Bestie supports OOP as a controlled and explicit toolset.

Core concepts:
	•	Classes (closed by default)
	•	Data / value / enum classes
	•	Single classes
	•	Protocols (static and virtual)
	•	Explicit inheritance rules
	•	Explicit polymorphism

Bestie intentionally avoids:
	•	Implicit virtual dispatch
	•	Inheritance ambiguity
	•	Fragile base classes

Full specification:

➡ oop.md — Object-Oriented Programming in Bestie

⸻

8. Memory Model (Overview)

Bestie uses manual, deterministic memory management with compile-time safety guarantees.

Core concepts:
	•	ptr<T>
	•	own<T>
	•	ref<T>
	•	Explicit allocation via .new()
	•	Explicit deallocation via .free() / .freeDeep()
	•	No garbage collection
	•	No null values
	•	No undefined behavior

Memory APIs reduce boilerplate without hiding behavior.

Full specification:

➡ memory.md — Memory Core & Memory API

⸻

9. Concurrency Model (Overview)

Concurrency in Bestie is:
	•	Explicit
	•	Compile-time validated
	•	Free of hidden synchronization
	•	Decoupled from memory ownership

Core primitives:
	•	OS threads
	•	Lightweight threads
	•	Explicit scheduling
	•	No implicit shared mutable state

High-level abstractions live in APIs, not the core.

Full specification:

➡ concurrency.md — Concurrency Core & API

⸻

10. Annotations and Plugins (Overview)

Annotations:
	•	Are resolved at compile time
	•	Introduce no runtime overhead
	•	Cannot mutate core semantics

Examples:
	•	@virtual
	•	@override
	•	@immutable
	•	@inline
	•	@expose

Custom annotations require compiler plugins.

Full specification:

➡ annotation.md — Annotations & Compiler Plugins

⸻

11. Error Handling (Overview)

Bestie does not use:
	•	Null
	•	Nil
	•	Undefined
	•	Implicit exceptions

Error handling mechanisms:
	1.	Complete return
	2.	Partial return (?)
	3.	option<T>
	4.	Explicit error returns (Zig-style)

Errors are explicit and enforced by the compiler.

➡ errors.md

⸻

12. What Is Intentionally Not in Core

Deliberately excluded:
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
	3.	Preserve performance
	4.	Never become “don’t use this” advice

⸻

13. Specification Stability
	•	The core is sealed
	•	Behavior is explicit
	•	Backward compatibility is a priority
	•	Advanced features evolve through APIs and frameworks

⸻

14. Next Documents
	•	oop.md — Classes, Protocols, Inheritance, Polymorphism
	•	fp.md — Functions, Lambdas, Partial Functions
	•	memory.md — Ownership, Pointers, Allocation
	•	concurrency.md — Threads and Scheduling
	•	annotation.md — Compile-Time Extensions
	•	errors.md — Error Model
	•	effective-bestie/ — Best Practices & Guidelines

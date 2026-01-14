Bestie Core Language Specification

This document defines the Bestie Core Language.

The core is intentionally:
	•	Small
	•	Sealed
	•	Performance-critical
	•	Stable by design

Everything outside this document belongs to std-lib, std-api, or std-framework.

⸻

1. Core Philosophy

Bestie is designed around the following principles:
	1.	Native performance comparable to C, Zig, and Rust
	2.	No garbage collection
	3.	No null, none, or undefined
	4.	No unsafe blocks
	5.	Explicit ownership and lifetime awareness
	6.	Compile-time enforcement over runtime checks
	7.	No features users are told “not to use”
	8.	OOP, FP, and procedural styles are tools, not paradigms

Bestie is both a system programming language and a backend language at its core.

⸻

2. Primitive Types

Bestie provides a fixed set of primitive types:
	•	int32, int64
	•	uint32, uint64
	•	float32, float64
	•	bool
	•	byte
	•	char
	•	str
	•	void

Rules:
	•	All primitives are value types
	•	Always inline
	•	Never nullable
	•	Passed by value
	•	No implicit heap allocation

⸻

3. Variables

Bestie provides two variable declarations:
	•	val — immutable
	•	var — mutable

Rules:
	•	val is immutable by default
	•	var is mutable
	•	Global var is prohibited
	•	Global val is allowed but must be immutable
	•	No const keyword exists

Immutability in Bestie is structural, not keyword-based.

⸻

4. Class Kinds

Bestie supports the following class kinds:
	•	Data classes (data class)
	•	Value classes (value class)
	•	Enum classes (enum, enum class)
	•	Single classes (single class)
	•	Closed classes (class)
	•	Open classes (open class)
	•	Abstract classes (abstract class)

Rules:
	•	data, value, and enum classes are:
	•	Immutable
	•	Inline
	•	Header-less
	•	single defines exactly one instance
	•	class is closed by default
	•	open class allows inheritance
	•	abstract class allows partial implementation

⸻

5. Sealed Classes

Sealed classes restrict inheritance to the same module.
sealed class Result

Rules:
	•	All subclasses must be known at compile time
	•	Exhaustive handling is enforced
	•	Enables safe pattern matching
	•	Commonly used for error and state modeling

⸻

6. Protocols

Protocols define behavior without state.

protocol Serializable {
    fun serialize(): str
}

Rules:
	•	No fields
	•	No state
	•	No constructors
	•	Only method signatures
	•	Multiple protocols may be implemented

Protocols express capability, not structure.

⸻

7. Functions

7.1 Function Declaration
fun add(x: int, y: int): int {
    return x + y
}

Expression form:
fun add(x: int, y: int) = x + y

Rules:
	•	Return types may be inferred
	•	Expression bodies omit return
	•	No hidden allocations
	•	No implicit heap promotion

⸻

8. Lambdas

Lambdas use => syntax.
val sum = (x: int, y: int) => x + y

Rules:
	•	Compile-time resolved
	•	Inlined by default
	•	No implicit heap allocation
	•	No implicit capture

⸻

9. Return Semantics (Core Rule)

Bestie defines four explicit return kinds.

This is a core language guarantee.

⸻

9.1 Complete Return

The function always returns a value.

fun getUser(): User {
    return user
}
Caller usage:
val u = getUser()
Always allowed.

⸻

9.2 Partial Return (?)

The function may return a value or return nothing.
fun getUser()? : User {
    if (found) return user
}

Rules:
	•	The function must be marked with ?
	•	Direct assignment is forbidden

Invalid:
val u = getUser()   // compile error

Valid:
if (getUser()) {
    val u = it
}
Partial behavior is explicit and visible in APIs.

⸻

9.3 Option Type

Defined in std-lib:
	•	Option<T>

States:
	•	Present(T)
	•	NotPresent

Rules:
	•	Used when values must be stored, passed, or composed
	•	Returning Option should be rare but supported

⸻

9.4 Error Return

Errors are part of the type system.
fun readFile(): File | IOError

Rules:
	•	Errors must be handled or propagated
	•	No exceptions
	•	No hidden control flow

⸻

10. Annotations

Annotations are compile-time only.
@get("/users")
fun listUsers(): list<User>

Rules:
	•	Core annotations are limited and sealed
	•	Custom annotations require compiler plugins
	•	No runtime reflection
	•	No runtime overhead

The compiler understands annotations only if:
	•	They are built into the core, or
	•	A plugin explicitly handles them

⸻

11. Plugins

Plugins:
	•	Operate on compiler IR
	•	Cannot modify core semantics
	•	Are sandboxed
	•	Cannot mutate compiler internals
	•	Cannot inject unsafe behavior

This prevents malicious or unstable extensions.

⸻

12. Concurrency (Core Level)

The core defines execution primitives only.
	•	OS threads
	•	Lightweight threads

Rules:
	•	No shared mutable state in core
	•	No mutex or atomic in core
	•	Ownership transfer enforces safety
	•	Concurrency libraries live outside the core

⸻

13. Explicit Exclusions

The core intentionally does not include:
	•	Garbage collection
	•	Unsafe blocks
	•	Reflection
	•	Macros
	•	Dependency injection
	•	IO abstractions
	•	Framework logic

These belong to higher layers.

⸻

14. Stability Guarantee

The core is conservative by design.
	•	Changes are rare
	•	Breaking changes require major versions
	•	The core is designed to survive future hardware evolution

Bestie evolves around the core, not through it.

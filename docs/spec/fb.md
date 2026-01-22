Bestie Language — Functional Programming (FP)

This document defines Functional Programming constructs in Bestie.

Functional programming in Bestie is:
	•	Explicit
	•	Compile-time driven
	•	Allocation-aware
	•	Side-effect explicit
	•	Fully compatible with OOP and systems programming

FP in Bestie is not a separate paradigm.
It is a set of composable tools that integrate seamlessly with the core language.

⸻

1. FP Philosophy

Bestie rejects:
	•	Hidden closures
	•	Implicit heap allocation
	•	Lazy evaluation by default
	•	Runtime-only abstractions
	•	Magical type inference

Bestie enforces:
	•	Explicit data flow
	•	Compile-time resolution
	•	Ownership-aware functions
	•	Deterministic execution
	•	Zero-cost abstractions

Golden Rule

If a function call, binding, or dispatch can be resolved at compile time, it must be.

⸻

2. Functions

Functions are first-class values, but not implicitly heap-allocated.

2.1 Function Declaration

fun add(a: int, b: int): int {
    return a + b
}

Properties:
	•	Static dispatch by default
	•	No implicit captures
	•	No hidden allocation
	•	Explicit return types

⸻

2.2 Expression Functions

Single-expression functions may omit braces:

fun square(x: int): int = x * x


⸻

3. Lambdas

Lambdas are anonymous functions with explicit capture rules.

val f = (x: int): int => x * 2

Rules:
	•	Parameter types are explicit
	•	Return type inferred from body
	•	Captures must be explicit
	•	No implicit heap allocation

⸻

3.1 Capture Rules

By default, lambdas capture nothing.

Explicit capture syntax is required:

val factor: int = 3
val f = [factor](x: int) => x * factor

Rules:
	•	Captured values are copied
	•	own values cannot be captured
	•	Captures are immutable
	•	Capture layout is compile-time known

⸻

4. Higher-Order Functions

Functions may accept or return other functions.

fun apply(f: (int) -> int, x: int): int {
    return f(x)
}

Rules:
	•	Function types are compile-time types
	•	No runtime boxing
	•	No dynamic dispatch unless explicitly annotated

⸻

5. Partial Functions

Bestie supports partial functions, denoted by ?.

fun parseInt(s: str): int?

Rules:
	•	Caller must handle partiality
	•	Compiler enforces exhaustiveness
	•	No implicit exceptions

⸻

6. Option and Error-Oriented FP

Bestie avoids exceptions and nulls.

Preferred FP-style returns:
	•	option<T>
	•	T?
	•	Error returns

Example:

fun findUser(id: int): option<User>

Rules:
	•	Errors are values
	•	Control flow is explicit
	•	No hidden stack unwinding

⸻

7. Immutability in FP

FP in Bestie strongly prefers immutability.

Guidelines:
	•	Use val by default
	•	Prefer value types
	•	Avoid var in functional code
	•	Favor transformation over mutation

val users = users.map(u => u.withName("Alice"))

Immutability is enforced by:
	•	Type system
	•	Ownership rules
	•	Compile-time checks

⸻

8. Extension Functions

Bestie supports extension functions, similar in spirit to Kotlin, but with stricter compile-time guarantees.

Extension functions allow adding behavior to existing types without modifying them and without runtime cost.

⸻

8.1 Declaring Extension Functions

Syntax:

fun TypeName.functionName(params): ReturnType

Example:

fun str.isEmpty(): bool {
    return this.length == 0
}

Usage:

val s: str = "hello"
val empty = s.isEmpty()


⸻

8.2 Compilation Model

Extension functions are:
	•	Statically resolved
	•	Compiled as plain functions
	•	Desugared at compile time

The above call is equivalent to:

isEmpty(s)

There is:
	•	No virtual dispatch
	•	No vtables
	•	No runtime lookup
	•	No modification of the original type

⸻

8.3 this in Extension Functions

Inside an extension function:
	•	this refers to the receiver parameter
	•	this is immutable unless the receiver type allows mutation
	•	Resolution is compile-time

fun Point.magnitude(): float {
    return sqrt(this.x * this.x + this.y * this.y)
}


⸻

8.4 Extension Functions vs Member Functions

Rules:
	•	Member functions always win over extensions
	•	No override is possible
	•	No polymorphism through extensions

class A {
    fun f(): int
}

fun A.f(): int   // ❌ illegal (name collision)

This prevents ambiguity and preserves compile-time determinism.

⸻

8.5 Extension Functions and Protocols

Extension functions do not participate in protocol dispatch.

protocol Printable {
    fun print(): str
}

fun Printable.debug(): str {
    return "debug: " + this.print()
}

Rules:
	•	Extensions are not protocol implementations
	•	They cannot satisfy protocol requirements
	•	They are resolved statically at call site

⸻

8.6 Generic Extension Functions

Extensions may be generic:

fun <T> list<T>.head(): T? {
    return if (this.size > 0) this[0] else none
}

Rules:
	•	Fully monomorphized
	•	No type erasure
	•	No runtime overhead

⸻

9. Function Composition

Bestie supports explicit composition via functions.

fun compose<A, B, C>(
    f: (B) -> C,
    g: (A) -> B
): (A) -> C {
    return (x: A) => f(g(x))
}

Composition is:
	•	Explicit
	•	Type-safe
	•	Compile-time resolvable

⸻

10. Recursion

Recursion is supported but explicit.

Rules:
	•	No implicit tail-call optimization guarantee
	•	Tail recursion may be optimized by the compiler
	•	Stack usage is deterministic

fun fact(n: int): int {
    if (n <= 1) return 1
    return n * fact(n - 1)
}


⸻

11. FP and Memory Model

FP in Bestie respects ownership.

Rules:
	•	own<T> cannot cross function boundaries implicitly
	•	Passing ownership must be explicit
	•	Closures cannot capture owning references

This ensures FP remains compatible with:
	•	Manual memory management
	•	Concurrency guarantees

⸻

12. FP vs OOP Interoperability

FP and OOP are fully interoperable.
	•	Methods are functions with receivers
	•	Extension functions bridge FP and OOP
	•	Protocols define behavior contracts
	•	Lambdas replace many classic OO patterns

FP does not replace OOP.
It reduces accidental complexity.

⸻

13. What Bestie Deliberately Avoids in FP
	•	Implicit currying
	•	Lazy evaluation by default
	•	Runtime monads
	•	Effect systems hidden from the type system
	•	Reflection-based functional dispatch

⸻

14. Summary

Functional Programming in Bestie is:
	•	Explicit
	•	Compile-time resolvable
	•	Allocation-aware
	•	Ownership-safe
	•	Zero-cost by design

FP in Bestie exists to compose behavior clearly, not to obscure execution.

⸻

Next Documents
	•	oop.md — Object-Oriented Programming
	•	core.md — Language Core
	•	memory.md — Ownership & Allocation
	•	errors.md — Error Model
	•	effective-bestie/ — Practical FP & OOP Guidelines

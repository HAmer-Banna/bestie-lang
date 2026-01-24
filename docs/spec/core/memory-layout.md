Bestie Language — Memory Model & Ownership

This document defines Bestie’s memory model, ownership system, and layout guarantees.

Memory management in Bestie is:
	•	Manual
	•	Explicit
	•	Deterministic
	•	Compile-time validated
	•	Free of undefined behavior

Bestie does not attempt to hide memory.
It makes memory predictable, analyzable, and efficient.

⸻

1. Memory Philosophy

Bestie rejects:
	•	Garbage collection
	•	Implicit allocation
	•	Implicit sharing
	•	Hidden reference counting
	•	Runtime-only memory rules

Bestie enforces:
	•	Explicit ownership
	•	Explicit allocation and deallocation
	•	Compile-time lifetime reasoning
	•	Zero-cost abstractions
	•	Optimal memory layout

Golden Rule

If memory behavior can be resolved at compile time, it must be resolved at compile time.

⸻

2. Core Memory Types

Bestie defines three fundamental memory-related types:

Type	Meaning
own<T>	Unique ownership
ref<T>	Borrowed reference
ptr<T>	Raw pointer (explicit, controlled)

Each has distinct guarantees and restrictions.

⸻

3. own<T> — Ownership

own<T> represents exclusive ownership of a value or allocation.

val own user: User = User.new()

Properties:
	•	Exactly one owner
	•	Owner is responsible for deallocation
	•	Ownership transfer must be explicit
	•	No implicit copying

Rules:
	•	own<T> cannot be copied
	•	own<T> cannot be captured by lambdas
	•	own<T> cannot be aliased

3.1 own and const

own<T> is not allowed with const.

const own user: User   // ❌ illegal

Reason:
	•	const requires compile-time resolution
	•	Ownership implies runtime allocation and lifetime
	•	These concepts are fundamentally incompatible

⸻

4. ref<T> — Borrowed Reference

ref<T> represents a non-owning, borrowed view into an existing value.

fun printUser(user: ref<User>) {
    print(user.name)
}

Properties:
	•	No ownership
	•	No deallocation responsibility
	•	Lifetime is statically checked
	•	Cannot outlive the owner

Rules:
	•	ref<T> cannot be stored beyond its scope
	•	ref<T> cannot be returned unless explicitly allowed
	•	ref<T> cannot be captured by escaping lambdas

4.1 ref and const

ref<T> is not allowed with const.

const ref config: Config   // ❌ illegal

Reason:
	•	const has no runtime lifetime
	•	ref implies a borrow from a runtime value

⸻

5. ptr<T> — Raw Pointer

ptr<T> represents a raw memory address.

val p: ptr<int> = someInt.address()

Properties:
	•	No ownership semantics
	•	No lifetime guarantees
	•	Explicit dereferencing
	•	Explicit mutation

ptr<T> exists to:
	•	Interoperate with low-level APIs
	•	Enable explicit pass-by-reference
	•	Support systems-level programming

⸻

5.1 ptr and const

If a value is const, its pointer is also const.

const PI: float64 = 3.14
val p = PI.address()    // ptr<const float64>

Rules:
	•	ptr<const T> is read-only
	•	Mutation through such a pointer is forbidden
	•	This is enforced at compile time

This guarantees:
	•	No backdoor mutation of compile-time constants
	•	Full const-correctness

⸻

6. Pointer Dereferencing and Mutation

Bestie does not allow implicit dereferencing.

Reading:

val x = p.val()

Writing (explicit mutation):

p.val(42)

Rules:
	•	p.val() reads the pointed value
	•	p.val(newValue) writes the pointed value
	•	Write access is checked against constness

This model:
	•	Makes mutation explicit
	•	Avoids hidden aliasing
	•	Preserves analyzability

⸻

6.1 Passing by Reference Using ptr

ptr<T> is the canonical mechanism for pass-by-reference.

fun increment(p: ptr<int>) {
    p.val(p.val() + 1)
}

Rules:
	•	Caller controls aliasing
	•	Callee cannot assume ownership
	•	Mutation is explicit and visible

This avoids:
	•	Hidden reference parameters
	•	Implicit mutability
	•	ABI ambiguity

⸻

7. Allocation and Deallocation

7.1 Allocation

Allocation is always explicit:

val own user = User.new()

Rules:
	•	.new() allocates
	•	init() never allocates
	•	Allocation site is always visible

⸻

7.2 Deallocation

Deallocation is explicit:

user.free()

Or recursively:

user.freeDeep()

Rules:
	•	Double free is impossible by construction
	•	Use-after-free is statically prevented where possible
	•	Remaining cases are runtime-checked

⸻

8. Stack vs Heap

Bestie prefers stack allocation whenever possible.

Rules:
	•	Value types default to stack allocation
	•	Escape analysis promotes stack allocation
	•	Heap allocation requires .new()

The compiler is free to:
	•	Inline values
	•	Elide allocations
	•	Reorder layouts

As long as observable semantics are preserved.

⸻

9. Memory Layout Guarantees

Bestie always produces the best possible memory layout.

This is a language guarantee, not an optimization hint.

9.1 Identity Classes

For identity-bearing classes:
	•	Header size is minimized
	•	No unused metadata
	•	No hidden vtables unless required

⸻

9.2 Inlining

Inlining rules:
	•	Value classes are always inlineable
	•	Functions are inlined whenever safe
	•	Extension functions inline naturally

Inlining is applied whenever:
	•	It preserves semantics
	•	It improves layout or performance
	•	It does not violate explicit user intent

⸻

9.3 Padding and Alignment

Bestie guarantees:
	•	Optimal field ordering
	•	Minimal padding
	•	Correct alignment

Even if the user declares fields in a suboptimal order:

class A {
    b: int8
    x: int64
}

The compiler is allowed to reorder layout internally to:
	•	Minimize padding
	•	Preserve ABI guarantees

Logical order ≠ physical layout.

⸻

10. Memory and Concurrency

Memory safety is ownership-driven, not lock-driven.

Rules:
	•	own<T> cannot be shared across threads implicitly
	•	ref<T> cannot escape thread boundaries
	•	ptr<T> crossing threads is explicit and unsafe by design

Thread safety emerges from:
	•	Ownership
	•	Immutability
	•	Compile-time guarantees

⸻

11. What Bestie Deliberately Avoids
	•	Garbage collection
	•	Implicit reference semantics
	•	Shared mutable state by default
	•	Hidden synchronization
	•	Undefined behavior

⸻

12. Summary

Bestie’s memory model is:
	•	Explicit
	•	Deterministic
	•	Compile-time validated
	•	Layout-optimal
	•	Systems-grade

Memory in Bestie is not something you “hope works”.
It is something you reason about, control, and trust.

⸻

Related Documents
	•	core.md — Language Core
	•	oop.md — Object-Oriented Programming
	•	fp.md — Functional Programming
	•	concurrency.md — Threads & Scheduling
	•	errors.md — Error Model

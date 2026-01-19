Bestie Memory Model (memory.md)

This document defines the core memory model of Bestie.
It is one of the foundational pillars of the language and is intentionally strict.

Bestie is a native, no-GC, no-null, no-unsafe language with a memory model designed around:
	•	Minimal allocation
	•	Predictable lifetimes
	•	Explicit ownership
	•	Strong compile-time guarantees

⸻

1. Design Goals

Bestie’s memory model guarantees:
	1.	No memory leaks
	2.	No null references
	3.	No undefined behavior
	4.	No hidden allocations
	5.	No runtime ownership checks
	6.	No GC pauses
	7.	Best possible memory layout

All guarantees are enforced at compile time.

⸻

2. Memory Regions

Bestie uses three memory regions:

Region	Description
Stack	Default for values, locals, temporaries
Heap	Used only when ownership requires it
Arena	Optional, explicit, API-level


⸻

3. Core Memory Types

3.1 Value Types

The following are value types:
	•	int, float, bool, char
	•	str
	•	tuple
	•	enum
	•	value class
	•	data class (when fields are value or owned)

Characteristics:
	•	Stored inline
	•	Copied by value
	•	No headers
	•	No vtables
	•	No allocation by default

val x: int = 10
val p: Point = Point(1, 2)


⸻

3.2 ptr

ptr<T> is the only raw memory primitive.

val p: ptr<int>

Rules:
	•	Never nullable
	•	Never dangling
	•	Never implicitly dereferenced
	•	Arithmetic is explicit
	•	Lifetime is checked at compile time

⸻

3.3 dereferencing and value access

Bestie uses one unified rule:

p.val()

	•	.val() reads the value pointed to
	•	Works for both ptr<T> and ref T
	•	No *, no deref, no magic

val x: int = p.val()


⸻

4. Ownership Model

4.1 own

own T means exclusive ownership.

val own u: User = User.of(...)

Rules:
	•	Exactly one owner
	•	Owner is responsible for destruction
	•	Ownership can be moved, not copied
	•	Default for heap-allocated objects

⸻

4.2 ref

ref T means borrowed reference.

fun printUser(u: ref User) { ... }

Rules:
	•	Cannot outlive the owner
	•	Cannot free
	•	Cannot move ownership
	•	Zero runtime cost

⸻

4.3 Ownership Moves

val own a = User.of(...)
val own b = move a   // a is invalid after this

Compile-time enforced.

⸻

5. Binding vs Ownership

Keyword	Meaning
val	Immutable binding
var	Mutable binding
own	Ownership
ref	Borrow

These are orthogonal.

val own u: User
var ref r: User


⸻

6. Function & Lambda Memory Rules

6.1 Complete Functions

fun getUser(): User {
  return User.of(...)
}

	•	Must return in all paths
	•	No empty return
	•	No implicit default

⸻

6.2 Partial Functions

fun getUser()? : User {
  if (exists) return user
}

Rules:
	•	? is mandatory
	•	Caller must handle absence
	•	No empty value is ever returned

⸻

6.3 Lambda Capture Rules

val f = () => {
  print(x)
}

Rules:
	•	Captured values are copied
	•	Captured refs must outlive the lambda
	•	Captured owns are forbidden unless moved

⸻

7. Automatic Destruction Rules

7.1 Scope-Based Destruction

Everything allocated inside a function or lambda:
	•	Is automatically destroyed
	•	In reverse order
	•	Without user intervention

fun f() {
  val own u = User.of(...)
} // destroyed here


⸻

7.2 free vs freeDeep

Bestie does not expose free() in core.

Destruction rules:
	•	Value types: no-op
	•	Owned objects: destructor
	•	Containers: recursive destruction

freeDeep() exists only in memory API, not core.

⸻

8. Containers & Memory

8.1 Immutable by Default

val l: list<int> = {1,2,3}

	•	Immutable
	•	Inline where possible
	•	Adding returns a new list

⸻

8.2 Mutable via Builders

val l = list<int>.of(1,2,3)
l.add(4) // returns new list


⸻

8.3 Containers of Pointers / Objects

list<own User>
list<ptr<int>>

Rules:
	•	list<own T> owns its elements
	•	Destruction is recursive
	•	No leaks by design

⸻

9. Copy Semantics

9.1 Shallow Copy

Default assignment:

val a = b

Rules:
	•	Value types: bitwise copy
	•	Own types: forbidden
	•	Ref types: copy reference

⸻

9.2 Deep Copy

Explicit only:

val c = b.copy()

Rules:
	•	Must be explicitly implemented
	•	Often provided by std-lib for containers
	•	No implicit deep copies

⸻

10. new(), init(), and Allocation

10.1 Allocation Rules

Bestie does not encourage new().
	•	Objects are created via factories
	•	Builders handle complex allocation
	•	Compiler decides layout and placement

User.of(...)
File.open(...)


⸻

10.2 new() and init()

Rules:
	•	new() and init() are restricted
	•	Not part of public APIs
	•	Can be blocked via annotations
	•	Enforced by compiler

This is documented in oop.md, not here.

⸻

11. malloc / realloc / calloc

Bestie does not expose raw allocators in core.

Instead:
	•	new() implicitly knows size
	•	Layout is compile-time known
	•	Reallocation is container-level

Low-level allocation helpers live in memory-api, not core.

⸻

12. Concurrency & Memory

Rules:
	•	own cannot be shared across threads
	•	ref requires lifetime & thread validation
	•	Immutable objects are thread-safe by design
	•	single, data, value, enum are thread-safe

⸻

13. What Bestie Explicitly Rejects

Bestie does not have:
	•	null
	•	none
	•	nil
	•	undefined
	•	unsafe
	•	GC
	•	Runtime ownership checks
	•	Implicit heap allocation

⸻

14. Summary

Bestie’s memory model is:
	•	Explicit
	•	Minimal
	•	Predictable
	•	Safe
	•	Fast

It gives system-level control without system-level footguns.

Bestie does not try to hide memory.
Bestie makes memory impossible to misuse accidentally.

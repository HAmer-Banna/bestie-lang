Bestie Concurrency Model (concurrency.md)

This document defines the core concurrency model of Bestie.

Concurrency is a first-class, low-level feature, designed to support:
	•	System programming
	•	Backend services
	•	High-performance workloads
	•	Predictable parallel execution

Bestie deliberately separates concurrency primitives (core) from concurrency conveniences (API).

⸻

1. Design Goals

Bestie concurrency is designed to be:
	1.	Explicit
	2.	Compile-time safe
	3.	Allocation-aware
	4.	Deterministic
	5.	Low-overhead
	6.	Composable
	7.	Minimal in syntax

There is no implicit sharing, no magic scheduling, and no runtime locking surprises.

⸻

2. Concurrency Core vs Concurrency API

2.1 Concurrency Core

The core provides:
	•	Threads
	•	Task execution
	•	Synchronization primitives
	•	Memory visibility guarantees

The core is:
	•	Small
	•	Sufficient
	•	Locked down

Anything that can be expressed using core primitives must not be added to core again.

⸻

2.2 Concurrency API

The API provides:
	•	Actors
	•	Thread pools
	•	Async helpers
	•	Pipelines
	•	Structured concurrency helpers

These exist only to reduce boilerplate, never to add new semantics.

⸻

3. Core Execution Units

3.1 threadOs

threadOs represents a native OS thread.
	•	Heavyweight
	•	Long-lived
	•	Used for system-level work

val t = threadOs.of(() => {
  work()
})

t.join()

Properties
	•	Maps 1:1 to OS threads
	•	Owns its stack
	•	Explicit lifecycle

⸻

3.2 threadLight

threadLight represents a lightweight execution unit.
	•	Scheduled
	•	Stack-managed
	•	Cheap to create

val t = threadLight.of(() => {
  compute()
})

t.await()

Properties
	•	Not bound to OS threads
	•	Designed for high concurrency
	•	Deterministic scheduling

⸻

3.3 Class Type

Both threadOs and threadLight are:
	•	closed classes
	•	not inheritable
	•	not value classes
	•	Created only via factories

Inheritance is not allowed in concurrency core types.

⸻

4. Execution Rules

4.1 No Implicit Sharing

The following is forbidden:

val own u = User.of(...)

threadOs.of(() => {
  use(u) // ❌ compile error
})

Ownership cannot cross threads.

⸻

4.2 Allowed Sharing

The following is allowed:
	•	Immutable values
	•	Data / value / enum classes
	•	single classes
	•	Explicitly synchronized memory

val cfg = Config.load()

threadLight.of(() => {
  read(cfg)
})


⸻

5. Synchronization Primitives (Core)

5.1 Mutex

val m = mutex.of()

m.lock()
critical()
m.unlock()

Rules:
	•	Explicit lock/unlock
	•	No implicit RAII magic
	•	API may offer scoped helpers

⸻

5.2 Atomic

val c = atomic<int>.of(0)

c.increment()

Rules:
	•	Lock-free
	•	Type-safe
	•	Limited to primitive/value types

⸻

5.3 Memory Visibility

Bestie guarantees:
	•	Sequential consistency within a thread
	•	Explicit synchronization required across threads
	•	No hidden fences

⸻

6. Thread Safety by Design

The following are 100% thread-safe by design:
	•	data class
	•	value class
	•	enum
	•	single class
	•	Classes annotated @immutable
	•	Effectively immutable classes

The following are user responsibility:
	•	Mutable classes
	•	Open classes
	•	Classes with shared mutable state

⸻

7. Factories, Not new()

Concurrency types cannot be created via:

threadOs.new()    // ❌
threadOs.init()   // ❌

Only factories are allowed:

threadOs.of(...)
threadLight.of(...)

This is enforced by the compiler and documented in oop.md.

⸻

8. Compile-Time Resolution

Concurrency in Bestie is compile-time resolved:
	•	Ownership crossing threads is detected
	•	Illegal sharing is rejected
	•	Lifetimes are validated
	•	No runtime checks are added

If it compiles, it is safe by construction.

⸻

9. Concurrency API (Overview)

The following do not belong to core but are provided by APIs:
	•	actor
	•	thread pools
	•	async pipelines
	•	parallel collections
	•	structured concurrency helpers

Example (API):

actor Counter {
  var count = 0

  fun inc() {
    count++
  }
}

This is built entirely on core primitives.

⸻

10. No Weird Syntax

Bestie explicitly avoids:
	•	<-
	•	async/await keywords
	•	Implicit futures
	•	Hidden continuations

Concurrency is explicit, readable, and debuggable.

⸻

11. Error Handling & Concurrency

Errors:
	•	Do not cross thread boundaries implicitly
	•	Must be handled or propagated explicitly
	•	Follow the same rules as synchronous code

⸻

12. What Bestie Does NOT Have

Bestie does not have:
	•	Green threads hidden behind syntax
	•	Implicit async
	•	Data races by default
	•	Runtime schedulers making decisions for you
	•	Concurrency macros

⸻

13. Summary

Bestie concurrency is:
	•	Primitive but complete
	•	Safe without being restrictive
	•	Fast without being clever
	•	Explicit without being verbose

If you can write it in 50 lines using core primitives,
Bestie may offer a 5-line API — but never new semantics.

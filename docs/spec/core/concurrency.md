# Bestie Core Concurrency Model

This document defines the **core concurrency primitive** of Bestie.

Core concurrency in Bestie is:

* Low-level
* Deterministic
* Explicit
* Safe-by-default for `own` accounting, with first-class explicit low-level control via `ptr`

**Parallelism is a core feature**, available directly through `thread` — the language's 1:1 OS-thread primitive.

Cooperative fibers, channels, atomics, and locks live in **`bestie.lib.concurrency`**. They are helpers, not language structure. A program that never imports that library has no fiber scheduler and no extra concurrency runtime.

> Unlike Go/Java runtime models, Bestie does not implicitly create an extra `main` worker thread.
> Program entry starts on the process entry OS thread, and `main` executes there with `thread` semantics.

---

## 1. Design Principles

1. Minimal — one execution primitive in core
2. Explicit — no implicit scheduling or sharing
3. Compile-time validated ownership and sharing
4. Deterministic lifecycle
5. Low-overhead

**No implicit sharing, hidden scheduling, or runtime magic.**

---

## 2. OS Threads (`thread`)

`thread` maps 1:1 to an OS-level thread. It is the **parallel execution primitive**.

* Heavyweight, long-lived
* Mapped 1:1 to OS threads
* Owns its stack
* Fully deterministic lifecycle
* Runs in parallel on separate cores — **parallelism is explicit and guaranteed**

**Factory creation:**

```bestie
val t = thread.of(() => work())
```

**Available methods:**

| Method                 | Description                                               |
| ---------------------- | --------------------------------------------------------- |
| `join()`               | Waits until the thread finishes                           |
| `isAlive(): bool`      | Checks if the thread is still running                     |
| `id(): int`            | Returns thread identifier                                 |
| `interrupt()`          | Signals cancellation — must be handled explicitly in body |

This is the whole core surface: **spawn, observe, join, signal**. Scheduling policy — priority, affinity, stack size, naming — is platform-specific and lives in `bestie.api.os`, which is the layer allowed to track what each OS actually offers. `interrupt()` stays because it is the primitive `bestie.lib.concurrency`'s `CancellationToken` is built on.

---

## 3. Class Rules

`thread` is:

* A closed class
* Not inheritable
* Not a value class
* Created **only via factory**

```bestie
thread.new()      // ❌ forbidden
thread.init()     // ❌ forbidden
thread.of(...)    // ✅
```

No inheritance, reflection, or dynamic creation is allowed for the core thread type.

---

## 4. Ownership & Sharing

### 4.1 Forbidden

```bestie
val own u = User.new()
thread.of(() => use(u))    // ❌ compile error: implicit ownership sharing
```

### 4.2 Explicit Ownership Transfer

Ownership may be moved into a thread explicitly. The source binding becomes invalid immediately.

```bestie
val own u = User.new()
thread.of(move u, (own user: User) => use(user))   // ✅

use(u)    // ❌ compile error: moved value
```

### 4.3 Allowed — Immutable Sharing

Thread safety comes from the **type**, not the binding. `val` makes a binding immutable — it does not make the value deeply immutable or safe to share.

Safe to share across threads — deep immutability must be explicit:

* Primitive types (`int`, `float64`, `bool`, `char`, etc.)
* `data class`, `value class`, `enum` — value semantics by definition
* Classes annotated `@immutable`
* `const` values
* Immutable collections — built with `.immutable`: `list<T>.immutable.build()`

```bestie
val cfg = Config.load()                          // Config is a data class ✅
val xs  = list<int>.immutable.build()            // explicitly immutable ✅
thread.of(() => read(cfg))                       // ✅ safe
thread.of(() => process(xs))                     // ✅ safe
```

**Not** safe to share:

```bestie
val xs: list<int> = list<int>.build()
thread.of(() => read(xs))    // ❌ compile error: mutable list, val binding is not enough
```

---

## 5. Compile-Time Guarantees

* Implicit ownership sharing across threads is a **compile-time error**
* `own` cannot be implicitly shared; transfer with `move`
* `ptr<T>` may cross thread boundaries; races and lifetime are programmer-owned
* A `ref` field is not a thread-safe handle — sharing the object it names is done with `ptr<T>`

> If `own` sharing rules compile, ownership accounting is safe by construction.
> `ptr`-based sharing remains explicit programmer responsibility.

The same ownership rules apply to compiler-known spawn points in `bestie.lib.concurrency` (`fiber.of`). See that package.

---

## 6. Panic Behavior in Threads

A panic represents a violated invariant — the program is in an invalid state. There is no partial recovery.

**A panic in any thread terminates the entire program.**

There is no mechanism to catch or isolate a thread panic at the core level.

```bestie
val t = thread.of(() => {
    panic("something is wrong")   // entire program terminates
})
t.join()
```

### Thread Bodies Return `void`

Core thread bodies return `void`. There is no return value from a thread at the core level.

```bestie
thread.of(() => doWork())       // void body
```

### Communicating Results and Errors

To get results or typed errors (`!`) back from a worker thread, use **channels** from `bestie.lib.concurrency`:

```bestie
import bestie.lib.concurrency.Channel

val ch = Channel<int ! WorkError>.of(1)
thread.of(move ch, (ch) => {
    val outcome = try compute()
    ch.send(outcome)
})
val outcome = try ch.receive()
```

This keeps the core minimal and the result-passing pattern explicit.

---

## 7. Thread-Local Storage

`threadlocal` is a **storage modifier** — not a class, not a wrapper, not a container. It lives with the concurrency model because it is meaningful only in the presence of `thread`.

A `threadlocal` declaration gives each OS thread its **own independent copy** of the variable. Access is direct, with no `.get()`, `.set()`, or any other wrapper indirection.

```bestie
threadlocal val requestId: str = ""
threadlocal var callDepth: int = 0
```

### 7.1 Usage

Access and mutation are identical to any other variable:

```bestie
threadlocal var counter: int = 0

fun tick() {
    counter += 1
}
```

There is no ceremony. `counter` is a variable. The compiler routes reads and writes to the calling thread's copy.

### 7.2 Semantics

* Each `thread` gets its **own independent copy**, initialized from the declared initializer
* Fibers from `bestie.lib.concurrency` share the copy of the OS thread they run on
* Initialization is **per-thread at first access** (lazy, but compile-time lowered to a TLS slot — no heap allocation)
* The initializer expression must be a **compile-time constant** or a pure function of compile-time constants
* Copies are independent — writes in one thread are invisible to others

### 7.3 Scope

`threadlocal` is allowed at:

* **Module level** (top-level declaration)
* **Function level** (static local — initialized once per thread, not once per call)

`threadlocal` inside a regular expression or block (non-static context) is a **compile-time error**.

### 7.4 Rules

* `threadlocal val` — immutable per-thread binding (the value is constant for that thread's lifetime)
* `threadlocal var` — mutable per-thread binding
* The initializer must be compile-time constant
* No sharing across threads — compiler enforces this for `own` values
* `ptr` into a `threadlocal` from another thread is legal but programmer-responsibility (same as all `ptr` use)
* No runtime overhead beyond the platform TLS mechanism (hardware register + offset on x86_64/arm64)
* No `@threadlocal` annotation, no `ThreadLocal<T>` class — `threadlocal` is a first-class storage modifier

---

## 8. Memory Model

Whether two threads may touch the same memory, and what each is guaranteed to see, is a **language** question: no library can define it, because the answer constrains what the optimizer and the CPU are allowed to reorder. Core defines it here. `bestie.lib.concurrency` supplies the `atomic<T>` and `Lock` types that make use of it.

### 8.1 Data Race

A **data race** occurs when two threads access the same memory location, at least one access is a write, the accesses are not ordered by a happens-before edge (§8.2), and neither is an atomic operation.

**A data race is undefined behavior.** It is the one place in Bestie where the compiler will not save you, and it is reachable only through `ptr<T>` — the constructs that carry ownership (`own`, `ref` fields, moves) cannot produce one, because `own` cannot be implicitly shared (§4) and the compiler rejects the attempt.

This is the same deal as the rest of the `ptr` axis (`memory.md` §15): the boundary is explicit in source, and past it the invariant is yours.

### 8.2 Happens-Before

Core guarantees the following edges. Everything written before the edge is visible to everything sequenced after it, with no explicit fence:

| Edge | Guarantee |
| ---- | --------- |
| **Program order** | Within one thread, effects appear in source order |
| **Spawn** | Everything the parent did before `thread.of(...)` happens-before the thread body begins |
| **Join** | Everything the thread body did happens-before `t.join()` returns |
| **Move** | A `move` across a spawn boundary carries the moved object's full prior state; the receiver observes it completely initialized |
| **Transitivity** | If A happens-before B and B happens-before C, then A happens-before C |

These edges are what make the ownership rules sound: transferring an `own` value into a thread needs no synchronization because the spawn edge already orders it.

Fibers run on a host `thread` and add no edges of their own — two fibers on the same scheduler are ordered by that thread's program order (`std-lib/concurrency.md` §3).

### 8.3 Atomic Ordering

An atomic operation carries an **ordering** that says which non-atomic accesses around it are constrained. Core fixes the vocabulary and the meanings; `bestie.lib.concurrency` provides `atomic<T>`, whose operations take one of these:

| Ordering | Meaning |
| -------- | ------- |
| `relaxed` | Atomic with respect to this location only. No ordering of other accesses. Safe for counters whose value is read later under some other edge |
| `acquire` | On a load: no subsequent access in this thread may be reordered before it. Pairs with a `release` store to give a happens-before edge |
| `release` | On a store: no prior access in this thread may be reordered after it. Pairs with an `acquire` load |
| `acqRel` | On a read-modify-write: `acquire` on the read, `release` on the write |
| `seqCst` | As above, plus a single total order over all `seqCst` operations that every thread agrees on |

**A `release` store observed by an `acquire` load of the same location creates a happens-before edge** between the storing and loading threads. This is the primitive every lock, channel, and lock-free structure is built from.

Bestie has **no default ordering**. `atomic<T>` operations name theirs, because a silently-`seqCst` default is a hidden cost on architectures where it means a fence, and a silently-`relaxed` default is a hidden bug. This is the same rule as everywhere else in the language: the cost is written down.

Atomic operations are never data races, whatever their ordering.

### 8.4 What Core Guarantees About Layout

* An access to a value of a primitive type, a `ptr<T>`, or any type at most one machine word wide and naturally aligned is a **single memory access**. The compiler will not split it into narrower accesses, and will not fabricate a write to a location the program did not write.
* Distinct fields of an object are distinct memory locations. Writing one field never writes another, so two threads touching two different fields of the same object do not race — field packing (`memory.md` §18.1) never merges independent fields into one access.
* Adjacent `bool` fields are byte-sized, not bit-packed, precisely so that this holds (`docs/compiler/compiler-architecture.md`).
* Values wider than a machine word have no atomicity guarantee. Sharing one across threads requires a `Lock`, or ownership transfer.

### 8.5 What Core Does Not Guarantee

* That a `ptr<T>` shared across threads is race-free — that is programmer-owned (§5, `memory.md` §14)
* Any ordering between accesses to *different* locations, absent an edge from §8.2 or a `release`/`acquire` pair
* Progress. A spin loop with no atomic operation and no yield may be optimized as the compiler sees fit; use `atomic<T>` or a `Lock`
* Anything about memory the program did not allocate — MMIO has its own rules (`std-api/memory.md` §5), and volatile semantics are **not** implied by any ordering above

---

## 9. What Core Does Not Include

Core provides `thread` and `threadlocal`. Everything else is **`bestie.lib.concurrency`**, built on top:

* Fibers (`fiber`)
* Channels
* Mutexes, atomics
* Cooperative cancellation tokens
* Thread pools and work-stealing schedulers

These are **library helpers** — not language features and not OS APIs.

Actors, structured concurrency, and reactive pipelines belong in frameworks or application code, not in core and not in this library's required surface.

---

## 10. Summary

| Need | Use |
| ---- | --- |
| Parallel execution on multiple cores | `thread` (core) |
| High-concurrency cooperative I/O | `fiber` (`bestie.lib.concurrency`) |
| Message passing between threads | `Channel` (`bestie.lib.concurrency`) |
| Shared mutable state | Locks / atomics (`bestie.lib.concurrency`) |

> `thread` is the only core execution primitive.
> Parallelism is in core. Fibers and coordination are in std-lib.

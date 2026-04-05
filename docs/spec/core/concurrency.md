# Bestie Core Concurrency Model

This document defines the **core concurrency primitives** of Bestie.

Core concurrency in Bestie is:

* Low-level
* Deterministic
* Explicit
* Safe-by-default for `own/ref` paths, with explicit unsafe boundaries via `ptr`

**Parallelism is a core feature**, available directly through `threadOs` without reaching into any API layer.

Higher-level concurrency patterns (actors, channels, structured concurrency, thread pools) live in **std-api** and are built on top of these two primitives.

> Unlike Go/Java runtime models, Bestie does not implicitly create an extra `main` worker thread.
> Program entry starts on the process entry OS thread, and `main` executes there with `threadOs` semantics.

---

## 1. Design Principles

1. Minimal — two primitives only
2. Explicit — no implicit scheduling or sharing
3. Compile-time validated ownership and sharing
4. Deterministic lifecycle
5. Low-overhead

**No implicit sharing, hidden scheduling, or runtime magic.**

---

## 2. Core Execution Units

### 2.1 OS Threads (`threadOs`) — Parallel Execution

`threadOs` maps 1:1 to an OS-level thread. It is the **parallel execution primitive**.

* Heavyweight, long-lived
* Mapped 1:1 to OS threads
* Owns its stack
* Fully deterministic lifecycle
* Runs in parallel on separate cores — **parallelism is explicit and guaranteed**

**Factory creation:**

```bestie
val t = threadOs.of(() => work())
```

**Available methods:**

| Method                 | Description                                               |
| ---------------------- | --------------------------------------------------------- |
| `join()`               | Waits until the thread finishes                           |
| `isAlive(): bool`      | Checks if the thread is still running                     |
| `priority(level: int)` | Suggests execution priority (optional, platform-specific) |
| `id(): int`            | Returns thread identifier                                 |
| `interrupt()`          | Signals cancellation — must be handled explicitly in body |

---

### 2.2 Lightweight Threads (`threadLight`) — Concurrent Execution

`threadLight` is a **cooperative lightweight thread** (fiber). It is the **concurrent execution primitive**.

* Low-overhead, high-concurrency
* Cooperatively scheduled — yields control explicitly
* Runs on an OS thread managed by the runtime scheduler
* Does **not** run in parallel with other `threadLight` instances on the same scheduler
* Deterministic lifecycle semantics

**Factory creation:**

```bestie
val t = threadLight.of(() => compute())
```

**Available methods:**

| Method                | Description                                    |
| --------------------- | ---------------------------------------------- |
| `await()`             | Waits until the light thread completes         |
| `isCompleted(): bool` | Checks if execution finished                   |
| `cancel()`            | Cancels execution (explicit handling required) |
| `id(): int`           | Returns internal thread identifier             |

**Yield points** — a `threadLight` yields at:

* `await()` calls on other light threads
* Explicit `threadLight.yield()`
* Blocking I/O calls (runtime parks the fiber and resumes when ready)

There is no preemptive scheduling. Execution is deterministic within a scheduler.

---

## 3. Parallel vs Concurrent — Core Distinction

| Primitive     | Model        | Scheduling    | Parallelism |
| ------------- | ------------ | ------------- | ----------- |
| `threadOs`    | OS thread    | Preemptive    | Yes — explicit, guaranteed |
| `threadLight` | Fiber        | Cooperative   | No — concurrent only |

Use `threadOs` when you need work to run **at the same time** on multiple cores.
Use `threadLight` when you need high-concurrency **I/O or coordination** without the cost of OS threads.

---

## 4. Class Rules

Both `threadOs` and `threadLight` are:

* Closed classes
* Not inheritable
* Not value classes
* Created **only via factories**

```bestie
threadOs.new()      // ❌ forbidden
threadLight.init()  // ❌ forbidden
```

```bestie
threadOs.of(...)    // ✅
threadLight.of(...) // ✅
```

No inheritance, reflection, or dynamic creation is allowed for core concurrency types.

---

## 5. Ownership & Sharing

### 5.1 Forbidden

```bestie
val own u = User.new()
threadOs.of(() => use(u))    // ❌ compile error: implicit ownership sharing
```

### 5.2 Explicit Ownership Transfer

Ownership may be moved into a thread explicitly. The source binding becomes invalid immediately.

```bestie
val own u = User.new()
threadOs.of(move u, (own user: User) => use(user))   // ✅

use(u)    // ❌ compile error: moved value
```

### 5.3 Allowed — Immutable Sharing

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
threadOs.of(() => read(cfg))                     // ✅ safe
threadOs.of(() => process(xs))                   // ✅ safe
```

**Not** safe to share:

```bestie
val xs: list<int> = list<int>.build()
threadOs.of(() => read(xs))    // ❌ compile error: mutable list, val binding is not enough
```

---

## 6. Compile-Time Guarantees

* Implicit ownership sharing across threads is a **compile-time error**
* `ref` crossing a thread boundary is a **compile-time error**
* Lifetimes are validated at the thread creation site
* No implicit runtime ownership checks for safe (`own/ref`) code paths

> If `own/ref` sharing rules compile, concurrency sharing is safe by construction.
> `ptr`-based sharing remains explicit programmer responsibility.

---

## 7. Panic Behavior in Threads

A panic represents a violated invariant — the program is in an invalid state. There is no partial recovery.

**A panic in any thread terminates the entire program.**

This applies to both `threadOs` and `threadLight`. There is no mechanism to catch or isolate a thread panic at the core level.

```bestie
val t = threadOs.of(() => {
    panic("something is wrong")   // entire program terminates
})
t.join()
```

### Thread Bodies Return `void`

Core thread bodies return `void`. There is no return value from a thread at the core level.

```bestie
threadOs.of(() => doWork())       // void body
threadLight.of(() => compute())   // void body
```

### Communicating Results and Errors

To get results or typed errors (`!`) back from a worker thread, use **std-api channels**:

```bestie
// std-api — not core
val ch = Channel<int ! WorkError>.new()
threadOs.of(move ch, (ch) => {
    val result = try compute()
    ch.send(result)
})
val result = ch.receive()
```

This keeps the core minimal and the result-passing pattern explicit.

---

## 8. What Core Does Not Include

Core provides the two execution primitives. Everything else is **std-api**, built on top:

* Channels (CSP-style message passing)
* Actors
* Mutexes, atomics, semaphores
* Thread pools and work-stealing schedulers
* Structured concurrency
* Supervised thread groups (isolation of panics at API level)
* Async helpers or parallel collections

These are **API-level patterns** — not language features.

---

## 8. Summary

| Need | Use |
| ---- | --- |
| Parallel execution on multiple cores | `threadOs` (core) |
| High-concurrency cooperative I/O | `threadLight` (core) |
| Message passing between threads | Channels (std-api) |
| Actor-based concurrency | Actors (std-api) |
| Shared mutable state | Locks / atomics (std-api) |

> `threadOs` and `threadLight` are the only core primitives.
> Parallelism is in core. Patterns are in std-api.

# Bestie Core Concurrency Model

This document defines the **core concurrency primitives** of Bestie.

Core concurrency in Bestie is:

* Low-level
* Deterministic
* Explicit
* Safe-by-default for `own/ref` paths, with explicit unsafe boundaries via `ptr`

All higher-level concurrency abstractions (actors, mutexes, channels, structured concurrency) live in **std-api**.

> Unlike Go/Java runtime models, Bestie does not implicitly create an extra `main` worker thread.
> Program entry starts on the process entry OS thread, and `main` executes there with `threadOs` semantics.

---

## 1. Design Principles

Core concurrency is designed to be:

1. Minimal
2. Explicit
3. Compile-time validated ownership/sharing
4. Deterministic
5. Low-overhead

**No implicit sharing, hidden scheduling, or runtime magic.**

---

## 2. Core Execution Units

### 2.1 OS Threads (`threadOs`)

`threadOs` represents a **true OS-level thread**:

* Heavyweight, long-lived
* Mapped 1:1 to OS threads
* Owns its stack
* Fully deterministic lifecycle
* Guaranteed parallelism (runs on separate cores when available)

**Factory creation:**

```bestie
val t = threadOs.of(() => work())
```

**Available methods:**

| Method                 | Description                                                   |
| ---------------------- | ------------------------------------------------------------- |
| `join()`               | Waits until the thread finishes                               |
| `isAlive(): bool`      | Checks if the thread is still running                         |
| `priority(level: int)` | Suggests execution priority (optional, platform-specific)     |
| `id(): int`            | Returns thread identifier                                     |
| `interrupt()`          | Requests thread cancellation (must handle explicitly in body) |

### 2.2 Lightweight Threads (`threadLight`)

`threadLight` represents **runtime-managed lightweight threads**:

* High-concurrency, cooperative scheduling
* Not bound to OS threads
* Opportunistic parallelism: runs in parallel if cores are available, otherwise concurrent
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

---

## 3. Class Rules

Both `threadOs` and `threadLight` are:

* Closed classes
* Not inheritable
* Not value classes
* Created **only via factories**

```bestie
threadOs.new()      // ❌ forbidden
threadLight.init()  // ❌ forbidden
```

> Only factories are allowed:

```bestie
threadOs.of(...)
threadLight.of(...)
```

No inheritance, reflection, or dynamic creation is allowed for core concurrency types.

---

## 4. Ownership & Sharing

### 4.1 Forbidden

```bestie
val own u = User.new()
threadOs.of(() => use(u)) // ❌ compile error
```

Ownership cannot be shared across threads implicitly.

Explicit ownership transfer is allowed only via `move`, where the source binding becomes invalid immediately.

Example (explicit transfer):

```bestie
val own u = User.new()
threadOs.of(move u, (own user: User) => use(user)) // ✅ transfer ownership to spawned thread

use(u) // ❌ compile error (moved value)
```

### 4.2 Allowed

* Immutable values (`val`)
* Data, value, and enum declarations
* Effectively Immutable Classes or Explicit by annotated `@immutable`

```bestie
val cfg = Config.load()
threadLight.of(() => read(cfg))  // ✅ safe
```

---

## 5. Parallelism

Parallelism in Bestie core:

* `threadOs` **always runs in parallel** on available cores
* `threadLight` runs **parallel when possible**, otherwise concurrent
* Compiler enforces **parallel safety at compile-time** for `own/ref` rules
* Two threads may run in parallel **only if ownership rules allow**

> **Parallelism is a property of ownership, not syntax.** No special `parallel` blocks or fork/join keywords exist.

---

## 6. Compile-Time Guarantees

* Implicit ownership sharing across threads is rejected
* Illegal sharing is a **compile-time error**
* Lifetimes are validated
* No implicit runtime ownership checks are added for safe (`own/ref`) code paths

> If `own/ref` sharing rules compile, concurrency sharing is safe by construction.
> Pointer-based sharing remains explicit programmer responsibility.

---

## 7. Explicit Syntax, No Magic

Core concurrency avoids:

* Implicit futures
* Hidden continuations
* Runtime scheduling heuristics

Concurrency is **explicit, readable, and debuggable**.

---

## 8. What Core Does Not Include

* Mutexes, atomics, channels
* Actors, thread pools, structured concurrency
* Async helpers or parallel collections

All of these are **API-level conveniences**, implemented on top of core primitives.

---

## 9. Summary

Bestie **core concurrency** is:

* Primitive but sufficient
* Minimal and deterministic
* Explicit without verbosity
* Safe-by-default with explicit unsafe boundaries

> `threadOs` and `threadLight` are **the only core primitives**.
> Parallelism is guaranteed when ownership allows, everything else lives in std-api.

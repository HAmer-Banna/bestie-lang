# Bestie Core Concurrency Model

This document defines the **core concurrency primitives** of Bestie.

Core concurrency in Bestie is:

* Low-level
* Deterministic
* Explicit
* Compile-time safe

All higher-level concurrency abstractions (actors, mutexes, channels, structured concurrency) live in the **API/stdlib**.

> Bestie does **not** start with a default thread. Even `main` runs in an explicit thread created by the user.

---

## 1. Design Principles

Core concurrency is designed to be:

1. Minimal
2. Explicit
3. Compile-time safe
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

`threadLight` represents **cheap, stack-managed threads**:

* High-concurrency, cooperative scheduling
* Not bound to OS threads
* Opportunistic parallelism: runs in parallel if cores are available, otherwise concurrent
* Deterministic scheduling

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
val own u = User.of(...)
threadOs.of(() => use(u)) // ❌ compile error
```

Ownership **cannot cross threads**.

### 4.2 Allowed

* Immutable values (`val`)
* Data, value, enum, single classes
* Classes annotated `@immutable`

```bestie
val cfg = Config.load()
threadLight.of(() => read(cfg))  // ✅ safe
```

---

## 5. Parallelism

Parallelism in Bestie core:

* `threadOs` **always runs in parallel** on available cores
* `threadLight` runs **parallel when possible**, otherwise concurrent
* Compiler enforces **parallel safety at compile-time**
* Two threads may run in parallel **only if ownership rules allow**

> **Parallelism is a property of ownership, not syntax.** No special `parallel` blocks or fork/join keywords exist.

---

## 6. Compile-Time Guarantees

* Ownership crossing threads is rejected
* Illegal sharing is a **compile-time error**
* Lifetimes are validated
* No runtime checks are added

> If it compiles, it is safe by construction.

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
* Safe by compile-time rules

> `threadOs` and `threadLight` are **the only core primitives**.
> Parallelism is guaranteed when ownership allows, everything else lives in API/stdlib.

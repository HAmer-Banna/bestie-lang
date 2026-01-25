# Bestie Core Concurrency Model

This document defines the **core concurrency primitives** of Bestie.

Concurrency in core is:

* Low-level
* Deterministic
* Explicit
* Compile-time safe

All higher-level abstractions live in the **API/stdlib**.

---

## 1. Design Principles

Core concurrency is designed to be:

1. Minimal
2. Explicit
3. Compile-time safe
4. Deterministic
5. Low-overhead

No implicit sharing, hidden scheduling, or runtime magic.

---

## 2. Core Execution Units

### 2.1 OS Threads (`threadOs`)

* Heavyweight, long-lived
* Mapped 1:1 to OS threads
* Owns its stack, explicit lifecycle

```bestie
val t = threadOs.of(() => work())
t.join()
```

### 2.2 Lightweight Threads (`threadLight`)

* Cheap, stack-managed units
* Scheduled independently of OS threads
* High concurrency, deterministic

```bestie
val t = threadLight.of(() => compute())
t.await()
```

---

## 3. Class Rules

Both `threadOs` and `threadLight` are:

* Closed classes
* Not inheritable
* Not value classes
* Created **only via factories**

```bestie
threadOs.new()   // ❌ forbidden
threadLight.init() // ❌ forbidden
```

> Only factories are allowed:

```bestie
threadOs.of(...)
threadLight.of(...)
```

No inheritance or reflection is allowed for core concurrency types.

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
* Data/value/enum/single classes
* Classes annotated `@immutable`

```bestie
val cfg = Config.load()
threadLight.of(() => read(cfg))  // ✅ safe
```

---

## 5. Compile-Time Guarantees

* Ownership crossing threads is rejected
* Illegal sharing is a compile-time error
* Lifetimes are validated
* No runtime checks are added

> If it compiles, it is safe by construction.

---

## 6. Explicit Syntax, No Magic

Core concurrency avoids:

* Implicit futures
* Hidden continuations
* Runtime scheduling magic

Concurrency is **explicit, readable, and debuggable**.

---

## 7. What Core Does Not Include

* Mutexes, atomics, or channels
* Actors, thread pools, structured concurrency
* Async helpers or parallel collections

All of these are **API-level conveniences**, implemented on top of core.

---

## 8. Summary

Bestie **core concurrency** is:

* Primitive but sufficient
* Minimal and deterministic
* Explicit without verbosity
* Safe by compile-time rules

> `threadOs` and `threadLight` are **the only core primitives**. Everything else lives in API/stdlib.

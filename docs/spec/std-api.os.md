# std-api.os — Operating System API

This document defines the **Bestie Standard OS API (`std-api.os`)**.

`std-api.os` provides **explicit, minimal, and structured access** to operating system services.
It is **not** a framework, **not** a runtime, and **not** a replacement for the core language.

The API is designed to:

* Serve backend services, system tools, and infrastructure software
* Support portability without hiding system reality
* Avoid over-engineering and overlapping abstractions

---

## 1. Scope and Non-Goals

### 1.1 What `std-api.os` Provides

* Process management
* Environment variables
* Signals
* Time and clocks (OS-level)
* Resource limits
* OS capabilities and metadata

---

### 1.2 What `std-api.os` Does *Not* Provide

* File I/O (see `std-api.io`)
* Memory management or MMIO (see `std-api.memory`)
* Networking (see `std-api.network`)
* CLI parsing (see `std-api.cli`)
* Concurrency primitives (core language / ext-concurrency)
* Framework-level abstractions

---

## 2. Design Principles

1. **Explicit over implicit**
2. **Classes for stateful OS concepts**
3. **Functions for stateless queries**
4. **No hidden background behavior**
5. **Portable surface, platform-specific backends**
6. **Zero overlap with core language**

---

## 3. Namespacing

All OS APIs live under:

```text
std.api.os
```

No re-exports. No wildcards.

---

## 4. Process Management

### 4.1 `Process`

Represents an OS process.

```bestie
class Process {
    pid: int
    fun wait(): ExitStatus
    fun kill(signal: Signal): void
}
```

Processes are:

* Explicitly created
* Explicitly waited upon
* Never implicitly detached

---

### 4.2 Creating Processes

```bestie
fun spawn(
    path: str,
    args: list<str>,
    env: map<str, str> = {},
): Process
```

Rules:

* No shell expansion
* No implicit PATH resolution unless requested
* Errors are explicit

---

## 5. Environment Variables

Stateless utilities:

```bestie
fun getEnv(key: str): str | NotFound
fun setEnv(key: str, value: str): void
```

No global mutable environment abstraction.

---

## 6. Signals

### 6.1 `Signal`

```bestie
enum Signal {
    Term,
    Kill,
    Int,
    Hup
}
```

Platform-specific signals may be conditionally available.

---

### 6.2 Signal Handling

```bestie
fun onSignal(sig: Signal, handler: fun(Signal): void): void
```

Rules:

* Handlers are non-capturing lambdas
* Execution context is explicit
* No async magic

---

## 7. Time and Clocks

### 7.1 OS Clock Access

```bestie
fun now(): TimePoint
fun monotonic(): Duration
```

Rules:

* No implicit time zones
* No global mutable clock

---

## 8. Resource Limits

```bestie
class ResourceLimit {
    current: int
    max: int
}
```

```bestie
fun getLimit(kind: ResourceKind): ResourceLimit
fun setLimit(kind: ResourceKind, limit: ResourceLimit): void
```

---

## 9. Platform Introspection

```bestie
fun platform(): PlatformInfo
```

Includes:

* OS family
* Architecture
* Endianness
* Capabilities

---

## 10. OOP vs Functions (Explicit Policy)

* **Classes** represent long-lived OS entities (Process, ResourceLimit)
* **Functions** represent queries or actions

This rule is intentional and enforced to prevent API fragmentation.

---

## 11. Error Model

All errors are:

* Typed
* Explicit
* Non-exceptional

No hidden retries. No silent fallbacks.

---

## 12. Stability and Evolution

* APIs are additive within a major version
* Platform-specific extensions are namespaced
* No duplication of abstractions

---

## 13. Summary

`std-api.os` is:

* Minimal
* Explicit
* Practical
* Predictable

It exposes OS power **without leaking OS chaos into the language**.

---

This document is **finalized**.

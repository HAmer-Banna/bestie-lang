# std-api.memory — Memory & MMIO API

This document defines the **Bestie Standard Memory API (`std-api.memory`)**.

This API enables:

* Bare-metal programming
* Kernel and driver development
* Memory-mapped I/O (MMIO)

Without compromising:

* Core language safety
* Ownership guarantees
* Portability

---

## 1. Core Principle

> **The core language provides `ptr<T>` as a mechanism.
> `std-api.memory` defines the policy and semantics.**

No hardware semantics exist in the core language.

---

## 2. Scope and Non-Goals

### 2.1 What This API Provides

* Memory regions
* Memory-mapped I/O
* Volatile access
* Explicit ordering
* Platform-aware memory operations

---

### 2.2 What This API Does *Not* Provide

* General-purpose allocation
* Garbage collection
* Implicit synchronization
* Pointer arithmetic helpers
* Unsafe escape hatches

---

## 3. Memory Regions

### 3.1 `MemoryRegion`

```bestie
class MemoryRegion {
    base: address
    size: int
}
```

Represents a **contiguous memory range**.

Rules:

* Immutable once created
* Alignment enforced at creation
* Ownership is explicit

---

## 4. MMIO Regions

### 4.1 `MmioRegion<T>`

```bestie
class MmioRegion<T> {
    fun read(offset: int): T
    fun write(offset: int, value: T): void
}
```

Properties:

* Typed access
* Volatile semantics
* Explicit offsets
* No implicit caching

Internally:

* Uses `ptr<T>`
* Applies compiler barriers

---

## 5. Volatile Semantics

MMIO access guarantees:

* No reordering
* No elision
* No speculative reads

These guarantees **do not apply** to raw `ptr<T>` usage.

---

## 6. Bare Metal Usage

On bare metal platforms:

* `std-api.memory` may be provided by the platform
* `std-api.os` may be absent
* MMIO is the primary hardware interaction mechanism

This allows:

* Firmware
* Bootloaders
* Device drivers

---

## 7. Safety Rules

1. No aliasing of MMIO regions
2. No sharing across threads without ownership transfer
3. No implicit synchronization
4. All side effects are explicit

Violations are compile-time errors where possible.

---

## 8. Platform-Specific Extensions

Platform-specific memory APIs:

* Must live under sub-namespaces
* Must not alter core semantics

Example:

```text
std.api.memory.arm
std.api.memory.x86
```

---

## 9. Relationship to Other APIs

| API              | Responsibility         |
| ---------------- | ---------------------- |
| Core language    | `ptr<T>`, ownership    |
| `std-api.memory` | MMIO, memory semantics |
| `std-api.os`     | Process & OS resources |
| `std-api.io`     | Streams & files        |

---

## 10. Intentional Limitations

This API intentionally avoids:

* Pointer arithmetic helpers
* Unsafe casting
* Hidden fences
* Architecture leakage into core

Power is available — **only when explicitly requested**.

---

## 11. Summary

`std-api.memory` enables:

* Low-level control
* Bare metal programming
* Hardware interaction

While preserving:

* Language integrity
* Safety
* Portability

---

This document is a **draft** and will evolve alongside platform backends.

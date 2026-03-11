# Bestie Standard Library — Allocators Package

This document defines the **allocators package** of the Bestie standard library.
These allocators provide explicit, deterministic allocation strategies for performance-sensitive code.

Allocators in Bestie are:

* Explicit
* Deterministic
* Composable
* Free of hidden fallback behavior
* Compatible with core ownership semantics

---

## 1. Design Goals

The allocators package exists to provide:

1. Predictable allocation cost
2. Visible lifetime boundaries
3. Alternative strategies without changing core language semantics
4. Debug visibility when needed

Allocators are **library-level tools**.
They do not change the core memory model.

---

## 2. Arena

`Arena` is a **value class** that provides region-based allocation.

It allows fast allocation and bulk deallocation with well-defined lifetime semantics.

### Construction

```bestie
val arena = Arena.of(1, MB)
```

* `size: int` — numeric size
* `unit: SizeUnit` — enum (`KB`, `MB`, `GB`)

### Allocation

```bestie
arena.add(42)
arena.add(listOfInts)
```

* `add(T)` allocates a single value
* `add(list<T>)` allocates a contiguous sequence

All allocations belong to the arena and share the same lifetime.

### Lifetime Control

```bestie
arena.reset()   // reuse memory
arena.release() // invalidate arena
```

* `reset()` clears all allocations but keeps the arena usable
* `release()` permanently invalidates the arena; further use is a compile-time error

### Rules

* Arenas do not move memory
* Arenas do not escape their owning scope
* Arenas do not participate in ownership transfer

---

## 3. FixedBuffer

`FixedBuffer` allocates from a fixed-capacity contiguous region.

It is useful when memory limits must be strict and known in advance.

### Construction

```bestie
val fixed = FixedBuffer.of(4, KB)
```

### Allocation

```bestie
val p = fixed.alloc(256) // Option<ptr<byte>>
```

Semantics:

* Success returns `Option.Present(ptr<byte>)`
* Capacity exhaustion returns `Option.Not_Present`
* No implicit heap fallback is allowed

### Lifetime Control

```bestie
fixed.reset()   // clears allocation cursor
fixed.release() // invalidates backing region
```

### Rules

* Capacity is fixed after creation
* Allocation order is deterministic
* No hidden reallocation or growth

---

## 4. Debug

`Debug` wraps another allocator and adds diagnostics.

It is intended for leak tracking, double-free detection, and usage reporting in development builds.

### Construction

```bestie
val arena = Arena.of(1, MB)
val debug = Debug.of(arena)
```

### Diagnostics

```bestie
debug.report()
debug.assertNoLeaks()
```

### Rules

* Debug instrumentation must not hide ownership boundaries
* Debug wrapper must preserve the wrapped allocator's allocation semantics
* Debug features add visibility, not policy changes

---

## 5. Relationship to Other Layers

* `std-lib.allocators` provides strategy-level allocators (`Arena`, `FixedBuffer`, `Debug`)
* `std-api.memory` provides platform and hardware memory APIs (MMIO, regions, volatile operations)
* Core language remains unchanged: ownership and explicit unsafe boundaries still define correctness

---

## Summary

The allocators package provides:

* Region allocation (`Arena`)
* Fixed-capacity allocation (`FixedBuffer`)
* Allocation diagnostics (`Debug`)

These allocators keep memory behavior explicit and deterministic without expanding core language complexity.

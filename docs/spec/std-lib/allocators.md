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

## 1.1 Class Kinds and Ownership Rationale

All three allocator types in this package are **`class`** — not `value class` or `data class`.

The reasoning applies uniformly:

* Each allocator has **mutable internal state** (an allocation cursor, a used-bytes counter, allocation records)
* Each allocator has **identity** — copying an arena would produce two objects that believe they own the same backing buffer, leading to double-free
* Each allocator **owns its backing memory** via a raw `ptr<byte>` that it is responsible for freeing through `release()`
* `value class` is copy-by-value and cannot contain `own` fields — neither property is compatible with a mutable memory region

Field ownership inside allocators uses raw `ptr<byte>` for the backing buffer. `ptr<T>` carries no ownership qualifier — the allocator itself is the owner and discharges the obligation through `release()`. All other internal fields (capacity, used bytes) are plain integer primitives with no qualifier.

---

## 2. Arena

`Arena` is a **`class`** that provides region-based allocation.

It allows fast allocation and bulk deallocation with well-defined lifetime semantics.

### Construction

```bestie
val arena = Arena.of(1, MB)
```

* `size: int` — numeric size
* `unit: SizeUnit` — enum (`KB`, `MB`, `GB`)

### Allocation

```bestie
val n = arena.add(42)         // ptr<int>
val xs = arena.add(listOfInts) // ptr<list<int>>
```

* `add(T)` allocates a single value and returns a `ptr<T>` into arena-owned storage
* `add(list<T>)` allocates a contiguous sequence and returns a `ptr<list<T>>` into arena-owned storage

All allocations belong to the arena and share the same lifetime.

### Lifetime Control

```bestie
arena.reset()   // reuse memory
arena.release() // invalidate arena
```

* `reset()` clears all allocations but keeps the arena usable
* `release()` permanently invalidates the arena; further safe use is rejected when statically provable

### Rules

* Arenas do not move memory after allocation
* Arenas do not escape their owning scope
* A `ptr<T>` returned by `add` is programmer-owned: use after `release()` is a dangling pointer — not a compile-time error
* Arenas do not participate in ordinary `own` transfer semantics

---

## 3. FixedBuffer

`FixedBuffer` is a **`class`** that allocates from a fixed-capacity contiguous region.

It is useful when memory limits must be strict and known in advance.

### Construction

```bestie
val fixed = FixedBuffer.of(4, KB)
```

### Allocation

```bestie
val p = fixed.alloc(256) // ptr<byte> ?
```

Semantics:

* Success returns a present `ptr<byte>`
* Capacity exhaustion returns absent
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

`Debug` is a **`class`** that wraps another allocator and adds diagnostics.

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

* `bestie.lib.allocators` provides strategy-level allocators (`Arena`, `FixedBuffer`, `Debug`)
* `bestie.api.memory` provides platform and hardware memory APIs (MMIO, regions, volatile operations)
* Core language remains unchanged: ownership and explicit unsafe boundaries still define correctness

---

## Summary

The allocators package provides:

* Region allocation (`Arena`)
* Fixed-capacity allocation (`FixedBuffer`)
* Allocation diagnostics (`Debug`)

These allocators keep memory behavior explicit and deterministic without expanding core language complexity.

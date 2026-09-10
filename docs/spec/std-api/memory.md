# bestie.api.memory — Memory & MMIO API

This document defines the **Bestie Standard Memory API (`bestie.api.memory`)**.

This API enables:

* Bare-metal programming
* Kernel and driver development
* Memory-mapped I/O (MMIO)

without compromising core language safety, ownership guarantees, or portability.

---

## 1. Core Principle

> **The core language provides `ptr<T>` as a mechanism.
> `bestie.api.memory` defines the policy and semantics.**

No hardware semantics exist in the core language. `ptr<T>` can address memory; it says nothing about whether that memory is a device register, whether a read has a side effect, or whether the compiler may cache the value. This package is where those facts live.

---

## 2. Scope and Non-Goals

### 2.1 What This API Provides

* Memory region descriptors
* Memory-mapped I/O with volatile, non-elidable access
* Explicit memory and device barriers
* Platform-aware memory operations

### 2.2 What This API Does *Not* Provide

* General-purpose allocation (see `bestie.lib.allocators`)
* Garbage collection
* Implicit synchronization or hidden fences
* Pointer arithmetic helpers — `ptr<T>` already has them (`core/memory.md` §8.5)
* An "unsafe" escape hatch — `ptr<T>` and `@trusted` are already explicit and searchable

---

## 3. Class Kinds and Ownership Rationale

| Type | Kind | Why |
| ---- | ---- | --- |
| `MemoryRegion` | `value class` | A base-and-size description of a range. No identity, no ownership — describing a region is not holding one. Copied inline into every call. |
| `MmioRegion<T>` | `class` | A live mapping of device registers. It **must not be aliased** (§7), and `own` is what enforces that: exactly one owner, moved rather than copied. |
| `Ordering` | `enum` | Reused from `bestie.lib.concurrency` — the vocabulary is core's (`core/concurrency.md` §8.3) and this package does not define a second one. |

`MemoryRegion` is a value and `MmioRegion<T>` is an owned resource, and the split is the whole safety story: you may pass a *description* of device memory freely, but the *mapping* has one owner.

---

## 4. Memory Regions

```bestie
value class MemoryRegion {
    base: uint          // physical or virtual address, target-width (core/lang.md §5.1)
    size: uint          // length in bytes
}
```

`base` is a `uint` — the pointer-width unsigned integer, the same type `p.addr` yields (`core/memory.md` §8.4.2). It is deliberately **not** a `ptr<T>`: a `MemoryRegion` describes an address range that may not be mapped yet, and a `ptr<T>` in safe Bestie always designates live storage.

Rules:

* A `MemoryRegion` is a description, not a claim. Constructing one maps nothing and grants no access.
* `base + size` must not wrap; a region that would is `MemoryError.InvalidRegion`.
* Alignment is checked when the region is **mapped** (§5), not when it is described.

---

## 5. MMIO Regions

```bestie
class MmioRegion<T> {
    fun read(index: int): T
    fun write(index: int, value: T): void
    fun region(): MemoryRegion
    fun count(): int
    fun free(): void
}

fun map<T>(region: MemoryRegion): own MmioRegion<T> ! MemoryError
```

`map<T>` establishes a typed mapping over a region. `index` is in **units of `T`**, not bytes — `read(2)` on an `MmioRegion<uint32>` reads the register at byte offset 8.

Rules:

* `map<T>` fails with `MemoryError.Misaligned` unless `region.base` satisfies `alignOf(T)`, and with `MemoryError.SizeMismatch` unless `region.size` is a whole multiple of `sizeOf(T)`.
* On a hosted platform `map<T>` requires privilege and fails with `MemoryError.PermissionDenied` without it. On bare metal it typically succeeds for any well-formed region.
* An out-of-range `index` **panics**, exactly as `array<T>` does (`core/types.md` §5). A register index outside the mapping is a violated invariant, not a recoverable condition.
* The mapping is `own`. It is released by `free()`, never implicitly.

### 5.1 Example — a UART driver

```bestie
import bestie.lib.concurrency.Ordering

const UART_BASE : uint = 0x1000_0000
const UART_SIZE : uint = 0x100

const REG_DATA   : int = 0      // byte offset 0
const REG_STATUS : int = 1      // byte offset 4
const TX_READY   : uint32 = 0x20

fun openUart(): own MmioRegion<uint32> ! MemoryError {
    return try map<uint32>(MemoryRegion.new(base: UART_BASE, size: UART_SIZE))
}

fun putc(uart: ptr<MmioRegion<uint32>>, c: byte): void {
    // Spin until the transmitter is ready. Each read reaches the device.
    while ((uart.val.read(REG_STATUS) & TX_READY) == 0) { }

    fence(Ordering.Release)              // prior writes land before this one
    uart.val.write(REG_DATA, c.toUInt32())
}
```

The `while` loop is the reason this API exists. A normal `ptr<uint32>` read in that position could be hoisted out of the loop by the optimizer — the compiler is entitled to assume memory it can see does not change on its own — producing an infinite loop. An `MmioRegion` read cannot be (§6).

---

## 6. Volatile Semantics

Every `MmioRegion.read` and `MmioRegion.write` is a **device access**, and the compiler treats it accordingly:

| Guarantee | Meaning |
| --------- | ------- |
| **Not elided** | A read whose value is discarded still happens; a write that is later overwritten still happens. Both may have side effects at the device. |
| **Not duplicated** | One `read` call is exactly one bus read. The compiler will not re-issue it to avoid a spill. |
| **Not reordered with other MMIO accesses** | Accesses through `MmioRegion` keep their program order relative to each other. |
| **Not speculated** | A read is never issued on a path that was not taken. |
| **Not widened or split** | A `read` on `MmioRegion<uint32>` is one 32-bit access, never two 16-bit accesses and never part of a 64-bit one. |

**These guarantees apply only to `MmioRegion`.** A raw `ptr<T>` gets none of them — `core/memory.md` §8 defines `ptr<T>` as ordinary memory access, and the optimizer treats it as such. That separation is deliberate: making every `ptr<T>` volatile would cost every program to serve a few.

**What volatile does *not* give you** is ordering against *ordinary* memory, or against another CPU. MMIO accesses are ordered relative to each other, and nothing more. Where a device write must not be observed before an ordinary memory write — a DMA descriptor written to RAM before a doorbell register is rung — an explicit barrier is required.

### 6.1 Barriers

```bestie
fun fence(o: Ordering): void
fun deviceFence(o: Ordering): void
```

* `fence` is a **CPU memory barrier**, with the ordering vocabulary of `core/concurrency.md` §8.3. It orders ordinary memory accesses.
* `deviceFence` additionally drains the write path to devices. On targets where the two are identical it lowers to the same instruction; on targets with a separate I/O ordering domain it does not.

There are **no implicit fences.** An `MmioRegion` access emits no barrier of its own, because a barrier per access is a cost every driver would pay and few need at every access. Ordering across domains is written where it is needed.

---

## 7. Safety Rules

1. **No aliasing of MMIO regions.** Two `MmioRegion` values covering overlapping addresses are a bug the type system prevents by construction: a mapping is `own`, so it cannot be copied, and `map<T>` fails with `MemoryError.AlreadyMapped` for a region already mapped in this program.
2. **No sharing across threads without ownership transfer.** `MmioRegion` follows the ordinary rules (`core/concurrency.md` §4) — `move` it into the thread that drives the device.
3. **No implicit synchronization.** See §6.1.
4. **All side effects are explicit.** A device access is a method call on a mapping, never a field read.

Rules 1 and 2 are compile-time errors, enforced by ownership rather than by anything specific to this package. Rule 1's cross-mapping check is a run-time failure from `map<T>`, because the compiler cannot know what a computed base address refers to.

---

## 8. Bare Metal Usage

On a freestanding target:

* `bestie.api.memory` may be supplied by the platform rather than by the host OS
* `bestie.api.os` is typically absent — there is no process, no environment, no signals
* MMIO is the primary hardware interaction mechanism

This is the **Tier 1** configuration of `platform.md` §8: core plus std-lib, no `bestie-project.toml`, no hosted I/O. `bestie.lib.concurrency` remains available, which is why `Ordering` is reused here rather than redefined.

---

## 9. Error Model

```bestie
errors MemoryError {
    InvalidRegion,      // base + size wraps, or size is zero
    Misaligned,         // base does not satisfy alignOf(T)
    SizeMismatch,       // size is not a whole multiple of sizeOf(T)
    AlreadyMapped,      // the region overlaps a live mapping
    PermissionDenied,   // hosted platform, insufficient privilege
    Unsupported         // the target has no MMIO facility
}
```

Out-of-range register indices **panic** rather than returning an error, matching `array<T>`: an index outside a mapping whose size you declared is a program bug.

---

## 10. Platform-Specific Extensions

Platform-specific memory APIs live under sub-namespaces and must not alter the semantics defined here:

```text
bestie.api.memory.arm
bestie.api.memory.x86
```

Cache maintenance, TLB operations, and architecture-specific barriers belong there — they have no portable meaning.

---

## 11. Relationship to Other Layers

| Layer | Responsibility |
| ----- | -------------- |
| Core language | `ptr<T>`, ownership, `sizeOf` / `alignOf` / `offsetOf` |
| `bestie.lib.concurrency` | `Ordering`, atomics |
| `bestie.lib.allocators` | General-purpose allocation |
| `bestie.api.memory` | MMIO, volatile access, barriers |
| `bestie.api.os` | Process and OS resources — absent on bare metal |

---

## 12. Summary

`bestie.api.memory` gives low-level code hardware access without leaking hardware semantics into the language:

* **`ptr<T>` is a mechanism; this package is the policy.** Volatile, non-elidable, non-reordered access is a property of `MmioRegion`, not of every pointer — so ordinary code pays nothing for it.
* **A description is a value; a mapping is owned.** `MemoryRegion` copies freely, `MmioRegion<T>` does not, and that is what makes the no-aliasing rule a compile-time property rather than a convention.
* **No implicit fences.** Ordering across the memory and device domains is written where it is needed, with the same `Ordering` vocabulary the rest of the language uses.
* **Out-of-range is a panic, misalignment is an error.** One is a program bug, the other is a condition a caller can genuinely handle.

Power is available — **only when explicitly requested.**

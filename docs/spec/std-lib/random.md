# std-lib.random — Pseudo-Random Number Generation

This document defines the **Bestie Standard Random Library (`std-lib.random`)**.

Random belongs to **std-lib**, not `std-api`, because:

* Pseudo-randomness is a deterministic, portable algorithm — not an OS primitive
* A seeded generator produces identical sequences on every platform
* **Cryptographically secure** and **OS entropy** sources are explicitly *not* part of this module — they live in `std-api.os`

Bestie randomness is:

* **Explicit** — every generator is an instance; there is no hidden global RNG
* **Deterministic** — a seed fully determines the sequence
* **Reproducible** — the same seed yields the same output everywhere
* **Allocation-free** — generator state is small and stack-friendly

---

## 1. Design Principles

1. **No global RNG.** There is no implicit `random()` free function backed by hidden state. A program that wants randomness must hold a generator.
2. **Seeds are explicit.** Reproducibility is the default; entropy must be requested deliberately.
3. **Deterministic PRNG only.** This module never silently reaches for OS entropy.
4. **Separation of concerns.** Insecure-but-fast PRNG here; secure randomness in `std-api.os`.
5. **Distributions are functions over a generator.** The generator produces raw bits; distributions interpret them.

---

## 2. Namespacing

All random APIs live under:

```text
std.lib.random
```

No re-exports. No wildcards. No global instance.

```bestie
import std.lib.random
```

---

## 3. Class Kinds and Ownership Rationale

| Type | Kind | Reasoning |
| ---- | ---- | --------- |
| `Rng` | `protocol` | Behavior contract for any generator; enables swapping algorithms |
| `Pcg32` | `class` | Holds mutable internal state (advances on each draw); has identity |
| `Xoshiro256` | `class` | Same reasoning — mutable, stateful, identity-bearing |
| `Seed` | `value class` | A fixed-width integer seed; inlined; no identity |
| `UniformInt` | `value class` | Immutable bounds only; pure interpreter over an `Rng` |
| `UniformFloat` | `value class` | Same — immutable bounds, no state |

**Why generators are `class`, not `value class`:**
A PRNG *mutates* its internal state on every draw. That mutation is the whole point — two draws from the "same" generator must differ. A `value class` would be copied on assignment, silently forking the stream and breaking reproducibility guarantees. Identity and mutability are therefore required, so generators are `class`.

**Why distributions are `value class`:**
`UniformInt` and `UniformFloat` hold only their immutable bounds. They carry no draw state; all state lives in the `Rng` passed to them. They are pure, copyable, and heap-free.

**Field ownership:** No field in this module carries an `own` or `ref` qualifier. Generator state is composed of primitive integers; seeds and distribution bounds are value types. There is no heap allocation anywhere in this library.

---

## 4. The `Rng` Protocol

Every generator implements a single low-level contract: produce uniformly distributed raw bits. Higher-level shaping is built on top.

```bestie
protocol Rng {
    // Next 32 uniformly distributed bits.
    fun nextBits32(): uint32

    // Next 64 uniformly distributed bits.
    fun nextBits64(): uint64
}
```

Rules:

* Implementations must be deterministic given their seed
* `nextBits32` / `nextBits64` are the only required methods
* Everything else (ranges, floats, shuffles) is derived

---

## 5. Generators

### 5.1 `Seed`

```bestie
value class Seed {
    value: uint64
}
```

* A fixed-width, reproducible seed
* Copied by value; no identity
* Two equal `Seed` values produce identical streams from the same algorithm

Factory constructors (free functions):

```bestie
fun Seed.of(value: uint64): Seed
```

> Entropy-derived seeds come from `std-api.os` (see §9). This module never reads entropy on its own.

---

### 5.2 `Pcg32`

A small, fast, statistically strong generator (PCG family). The recommended default.

```bestie
class Pcg32 {
    fun nextBits32(): uint32
    fun nextBits64(): uint64
    fun copy(): Pcg32          // explicit stream fork
}

impl Rng for Pcg32
```

Construction uses static factory methods (`@noNew`):

```bestie
val rng = Pcg32.fromSeed(Seed.of(0xCAFEBABE))
```

* Deterministic: same seed → same sequence
* Mutable: each `nextBits*` call advances internal state
* `copy()` makes stream forking **explicit** — there is no accidental duplication via assignment

---

### 5.3 `Xoshiro256`

A larger-state generator for callers who want a longer period.

```bestie
class Xoshiro256 {
    fun nextBits32(): uint32
    fun nextBits64(): uint64
    fun copy(): Xoshiro256
}

impl Rng for Xoshiro256
```

```bestie
val rng = Xoshiro256.fromSeed(Seed.of(42))
```

Same rules as `Pcg32`: deterministic, mutable, explicit forking.

---

## 6. Distributions

Distributions are **value types** that interpret an `Rng`. They hold bounds only; they never hold draw state. This keeps the stream owned by exactly one generator.

### 6.1 `UniformInt`

```bestie
value class UniformInt {
    val low: int64    // inclusive
    val high: int64   // exclusive
}
```

```bestie
fun UniformInt.of(low: int64, high: int64): UniformInt   // panics if low >= high

fun sample(d: UniformInt, rng: Rng): int64
```

* Unbiased (rejection sampling internally; no modulo bias)
* `low >= high` is an invariant violation → panic

### 6.2 `UniformFloat`

```bestie
value class UniformFloat {
    val low: float64    // inclusive
    val high: float64   // exclusive
}
```

```bestie
fun UniformFloat.of(low: float64, high: float64): UniformFloat

fun sample(d: UniformFloat, rng: Rng): float64
```

* Produces values in `[low, high)`
* `UniformFloat.of(0.0, 1.0)` is the canonical unit-interval distribution

---

## 7. Convenience Functions

Free functions operating on any `Rng`. They are thin shapers over the protocol — no hidden generator is created.

```bestie
fun nextInt(rng: Rng, low: int64, high: int64): int64       // [low, high)
fun nextFloat(rng: Rng): float64                            // [0.0, 1.0)
fun nextBool(rng: Rng): bool
fun shuffle<T>(rng: Rng, items: list<T>): void              // in-place Fisher–Yates
fun choice<T>(rng: Rng, items: list<T>): T | Empty          // Empty if list is empty
```

Rules:

* Every function takes the generator **explicitly** as its first argument
* `shuffle` mutates the list in place and is allocation-free
* `choice` returns a typed empty result rather than panicking on an empty list

---

## 8. Error Model

This module is almost entirely **non-fallible** by design:

* Misuse of bounds (`low >= high`) is an **invariant violation → panic**, not a recoverable error
* Empty inputs to `choice` return a typed `| Empty` result, not an error
* There are no I/O paths, so there is nothing to fail at runtime

There is therefore no `errors` block in `std-lib.random`. Fallibility belongs at the entropy boundary in `std-api.os`.

---

## 9. Relationship to Secure Randomness

`std-lib.random` is **not** suitable for cryptography, tokens, passwords, or anything security-sensitive. Its generators are fast and predictable by design — an attacker who observes enough output can reconstruct the state.

Secure randomness and OS entropy live in `std-api.os`:

```bestie
import std.api.os
import std.lib.random

// Seed a deterministic PRNG from a one-time entropy draw.
val seed = Seed.of(os.entropy64())          // os.entropy64 lives in std-api.os
val rng  = Pcg32.fromSeed(seed)
```

The boundary is intentional:

* `std-lib.random` — deterministic, reproducible, fast, **insecure**
* `std-api.os` — entropy and cryptographically secure bytes, **fallible**, OS-backed

---

## 10. Concurrency

* Generators are **not** thread-safe. Each `nextBits*` call mutates state.
* Sharing one generator across threads without synchronization is a data race.
* The intended pattern is **one generator per thread**, each seeded explicitly (e.g. base seed + thread index) to preserve reproducibility.

```bestie
val rng = Pcg32.fromSeed(Seed.of(baseSeed + threadIndex))
```

Distributions (`UniformInt`, `UniformFloat`) are immutable value types and are freely shareable; only the generator they are sampled against carries mutable state.

---

## 11. Intentional Non-Features

This library intentionally excludes:

* A global/default RNG instance
* An implicit `random()` free function with no generator
* Automatic entropy seeding (must be requested via `std-api.os`)
* Cryptographically secure generators
* Locale- or time-dependent behavior
* Implicit stream forking on copy (use `copy()` explicitly)

---

## 12. Summary

`std-lib.random` is:

* Explicit — generators are always passed by hand
* Deterministic — seeds fully define sequences
* Reproducible — identical output across platforms
* Heap-free — generator state and distributions allocate nothing

Generators (`Pcg32`, `Xoshiro256`) are the only classes — the types that require identity and mutable state. Seeds and distributions are value types. Security lives elsewhere.

> Randomness in Bestie is **reproducible by default and secure only by explicit choice**.

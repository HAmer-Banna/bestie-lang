# Operating System API

This document defines the **Bestie Standard OS API (`bestie.api.os`)**.

`bestie.api.os` provides **explicit, minimal, and structured access** to operating system services.
It is **not** a framework, **not** a runtime, and **not** a replacement for the core language.

The API is designed to:

* Serve backend services, system tools, and infrastructure software
* Support portability without hiding system reality
* Avoid over-engineering and overlapping abstractions

---

## 1. Scope and Non-Goals

### 1.1 What `bestie.api.os` Provides

* Process management
* Environment variables
* Signals
* Time and clocks (OS-level)
* Resource limits
* OS capabilities and metadata
* Entropy and cryptographically secure randomness

---

### 1.2 What `bestie.api.os` Does *Not* Provide

* File I/O (see `bestie.api.io`)
* Memory management or MMIO (see `bestie.api.memory`)
* Networking (see `bestie.api.network`)
* CLI parsing (see `bestie.api.cli`)
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
bestie.api.os
```

No re-exports.

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

## 10. Entropy and Secure Randomness

The operating system is the **only** source of true entropy. Pseudo-random number
generation lives in [`bestie.lib.random`](../std-lib/random.md); this section provides the
raw, cryptographically secure material that PRNGs may be seeded from and that
security-sensitive code must use directly.

### 10.1 Entropy Queries

Stateless functions backed by the OS CSPRNG (e.g. `getrandom`, `BCryptGenRandom`):

```bestie
fun entropy64(): uint64 ! EntropyError
fun secureBytes(out: list<byte>): void ! EntropyError
```

Rules:

* Output is **cryptographically secure** — suitable for keys, tokens, and nonces
* Each call draws fresh entropy; there is no internal cached state
* `secureBytes` fills the provided buffer in place (no allocation)
* Blocking vs non-blocking behavior is platform-defined but never silently degraded

### 10.2 Error Model

Entropy access is **fallible** because the OS source may be unavailable
(early boot, sandboxed environment, exhausted handle):

```bestie
errors EntropyError {
    Unavailable,    // no OS entropy source accessible
    WouldBlock      // non-blocking source not yet seeded
}
```

```bestie
val seed = entropy64() catch |err| { ... }
```

### 10.3 Relationship to `bestie.lib.random`

* `bestie.api.os` — true entropy, **fallible**, OS-backed, secure
* `bestie.lib.random` — deterministic PRNG, reproducible, fast, **insecure**

The intended bridge: draw a one-time seed here, then run a fast deterministic
generator from `bestie.lib.random`.

```bestie
import bestie.api.os
import bestie.lib.random

val seed = Seed.of(os.entropy64() catch |err| { ... })
val rng  = Pcg32.fromSeed(seed)
```

Security-sensitive randomness must use `secureBytes` directly and never a
`bestie.lib.random` generator.

---

## 11. OOP vs Functions (Explicit Policy)

* **Classes** represent long-lived OS entities (Process, ResourceLimit)
* **Functions** represent queries or actions

This rule is intentional and enforced to prevent API fragmentation.

---

## 12. Error Model

All errors are:

* Typed
* Explicit
* Non-exceptional

No hidden retries. No silent fallbacks.

---

## 13. Stability and Evolution

* APIs are additive within a major version
* Platform-specific extensions are namespaced
* No duplication of abstractions

---

## 14. Summary

`bestie.api.os` is:

* Minimal
* Explicit
* Practical
* Predictable

It exposes OS power **without leaking OS chaos into the language**.

---

This document is **finalized**.

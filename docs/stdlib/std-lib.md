# Standard Library (std-lib)

This document defines the **Bestie Standard Library (std-lib)**.

The standard library provides **foundational, portable, and deterministic functionality** built directly on top of the Bestie core language. It is **not a framework**, **not opinionated**, and **not magical**.

std-lib exists to:

* Enable real-world programs
* Provide canonical implementations
* Preserve predictability and performance
* Avoid fragmentation

---

## 1. Design Principles

The Bestie standard library follows these principles:

1. **Explicit over implicit**
2. **Deterministic behavior**
3. **No hidden allocation**
4. **No global mutable state**
5. **No runtime reflection**
6. **No dependency injection**
7. **No background threads**
8. **No magic defaults**

If a behavior is not visible in the API, it does not happen.

---

## 2. Scope of std-lib

std-lib provides:

* Core data structures (beyond language primitives)
* Algorithms
* Concurrency primitives
* Time and scheduling
* I/O foundations
* OS abstraction
* Numeric utilities
* String and text processing
* Error types and result utilities

std-lib does **not** provide:

* Web frameworks
* UI frameworks
* ORM or database layers
* Serialization frameworks
* Dependency injection
* Configuration systems
* Logging frameworks (only primitives)

These belong to **std-api** or **third-party libraries**.

---

## 3. Namespacing and Structure

std-lib is organized into **flat, explicit namespaces**.

Examples:

```text
std.lang
std.collections
std.concurrent
std.time
std.io
std.fs
std.net
std.math
std.text
std.os
std.error
```

Rules:

* No wildcard imports
* No implicit re-exports
* Namespaces are stable across versions
* Symbols are not aliased implicitly

---

## 4. Language-Level vs Library-Level

Some concepts appear in both **core language** and **std-lib**:

### 4.1 Core Language Provides

* `int32`, `int64`, `float32`, `bool`
* `str`
* `list`, `map`, `set`
* `threadOS`, `mutex`, `channel`
* `own`, `ref`, `ptr`
* `Result<T, E>` (type only)

### 4.2 std-lib Provides

* Implementations
* Algorithms
* Utilities
* OS bindings
* Extended variants

The core defines **what exists**.
std-lib defines **how it behaves**.

---

## 5. Data Structures (std.collections)

std-lib extends core collections with:

* Specialized containers
* Algorithms
* Iteration utilities

Examples:

* `list`
* `map`
* `set`
* `queue`
* `deque`
* `stack`
* `ringBuffer`

Rules:

* Collections are **generic**
* Allocation is explicit
* Ownership rules are enforced
* No hidden resizing strategies

Example:

```bestie
val users = list<own User>.withCapacity(128)
```

---

## 6. Algorithms (std.algorithms)

Algorithms are:

* Free functions
* Stateless
* Deterministic

Examples:

* `sort`
* `binarySearch`
* `map`
* `filter`
* `reduce`

Rules:

* No allocation unless explicitly requested
* Stable and unstable variants are distinct
* Parallel variants require explicit opt-in

---

## 7. Error Handling (std.error)

std-lib formalizes error handling without exceptions.

### 7.1 Result Type

```bestie
Result<T, E>
```

Properties:

* Value-based
* No stack unwinding
* No hidden control flow

Utilities include:

* Mapping
* Chaining
* Pattern matching helpers

### 7.2 Error Types

Errors are:

* Plain data
* Immutable
* Comparable
* Serializable

No error carries stack traces by default.

---

## 8. Concurrency (std.concurrent)

std-lib builds on core concurrency primitives.

Provides:

* Thread coordination utilities
* Schedulers
* Atomic operations
* Thread-safe collections (explicit)

Rules:

* No green threads
* No async runtime
* No hidden thread pools

Example:

```bestie
val pool = threadPool.withSize(4)
```

Concurrency is **opt-in and explicit**.

---

## 9. Time and Scheduling (std.time)

Provides:

* `Instant`
* `Duration`
* Monotonic clocks
* Sleep and timers

Rules:

* No wall-clock assumptions
* No global time mutation
* Deterministic APIs

---

## 10. I/O Foundations (std.io)

I/O is:

* Blocking by default
* Explicitly buffered
* Explicitly flushed

Provides:

* Streams
* Readers / Writers
* Byte buffers

No async I/O abstractions are provided here.

---

## 11. File System (std.fs)

Provides:

* Files
* Directories
* Paths
* Metadata

Rules:

* No implicit working directory changes
* Errors are explicit
* No global filesystem state

---

## 12. Networking (std.net)

Provides:

* TCP
* UDP
* Sockets
* Address handling

Rules:

* No HTTP abstraction
* No TLS abstraction
* No background event loops

These are built in **std-api** or external libraries.

---

## 13. Math and Numerics (std.math)

Provides:

* Mathematical constants
* Numeric utilities
* Deterministic algorithms

Examples:

```bestie
math.PI
math.sqrt(x)
```

No SIMD auto-vectorization is assumed.

---

## 14. Text and Strings (std.text)

Provides:

* String manipulation
* Encoding utilities
* Parsing helpers

Rules:

* `str` is UTF-8
* No implicit encoding conversion
* Explicit normalization

---

## 15. OS Abstraction (std.os)

Provides:

* Environment variables
* Process execution
* Signals
* Platform detection

Rules:

* Thin abstraction
* No policy
* No lifecycle management

---

## 16. Versioning and Stability

std-lib follows semantic versioning **independently**:

```text
bestie <lang>.<core>.<std-lib>.<std-api>
```

Rules:

* Breaking changes increment std-lib version
* APIs are stable within a major version
* Experimental modules are explicitly marked

---

## 17. What std-lib Explicitly Rejects

std-lib does not include:

* Reflection
* Runtime code generation
* Dependency injection
* Global registries
* Service locators
* Hidden background services

---

## 18. Summary

The Bestie standard library is:

* Small but complete
* Explicit and predictable
* Performance-oriented
* Safe by construction
* Free of hidden behavior

std-lib is a **toolbox**, not a framework.
It provides the **minimum necessary power** to build systems cleanly and deliberately.

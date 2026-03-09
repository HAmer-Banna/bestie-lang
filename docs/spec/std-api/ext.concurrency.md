# Extended Concurrency API (`std-api.ext.concurrency`)

This document defines the **Extended Concurrency API** for Bestie.

This API builds **strictly on top of the core concurrency primitives**:

* `threadOs`
* `threadLight` (fiber)
* Structured concurrency
* CSP-style message passing

The purpose of this API is to provide **specialized coordination tools** for advanced or domain-specific use cases, without polluting the core language.

---

## 1. Design Philosophy

The extended concurrency API follows these principles:

1. **No duplication of core semantics**
2. **No alternative concurrency models**
3. **Explicit over implicit**
4. **Specialized tools only**
5. **Zero magic**
6. **Composable with core ownership rules**

If a problem can be solved cleanly with:

* threads/fibers
* structured concurrency
* CSP

then **it must not introduce a new abstraction**.

---

## 2. What Lives in Core vs Extended API

### Core Concurrency (Already Sealed)

Provided by the language itself:

* OS threads (`threadOs`)
* Fibers (`threadLight`)
* Structured concurrency (implicit by ownership & lifetimes)
* CSP primitives (channels)
* Ownership transfer as synchronization
* Deterministic execution guarantees

This covers **90–95% of real-world concurrency needs**.

---

### Extended Concurrency API (This Document)

Provides **specialized primitives** only when core tools are insufficient.

Examples:

* Low-level atomic operations
* Explicit locking (when unavoidable)
* Cooperative cancellation
* Backpressure signaling

---

## 3. Atomic Operations

### 3.1 Purpose

Atomics exist for **low-level performance-critical scenarios** such as:

* Lock-free algorithms
* Runtime internals
* Specialized data structures

They are **not intended for general application logic**.

---

### 3.2 Atomic Types

```bestie
atomic<T>
```

Rules:

* `T` must be a primitive value type
* Operations are explicit
* No implicit memory ordering
* No hidden fences

Example:

```bestie
atomic<int> counter = atomic.new(0)
counter.increment()
```

---

### 3.3 Why Atomics Are Not Core

* Atomics introduce memory-ordering complexity
* They are rarely needed in business logic
* Ownership + CSP already solve most problems

Therefore atomics are **opt-in via API**, not language constructs.

---

## 4. Locks

### 4.1 Purpose

Locks exist **only when mutation must be shared** and ownership transfer is impractical.

They are a **last resort**, not a default tool.

---

### 4.2 Lock Types

```bestie
Lock<T>
```

Properties:

* Explicit acquire/release
* No implicit locking
* No reentrancy by default
* Ownership-aware

Example:

```bestie
lock.acquire()
updateSharedState()
lock.release()
```

---

### 4.3 Design Constraints

* Locks do not introduce shared ownership
* Locks do not bypass ownership rules
* Deadlock prevention is enforced by structure, not heuristics

---

## 5. Cancellation

### 5.1 Motivation

Cancellation is required for:

* Long-running fibers
* Cooperative task shutdown
* Graceful service termination

---

### 5.2 Cancellation Token

```bestie
CancellationToken
```

Rules:

* Explicit propagation
* Cooperative only
* No forced termination
* No async exceptions

Example:

```bestie
fun worker(token: CancellationToken): void {
    while not token.cancelled() {
        doWork()
    }
}
```

---

### 5.3 Why Cancellation Is API-Level

* Core threads/fibers remain minimal
* Cancellation policies vary widely
* Avoids implicit runtime behavior

---

## 6. Backpressure

### 6.1 Purpose

Backpressure is required in:

* Streaming systems
* Networking
* IO pipelines
* Reactive workloads

---

### 6.2 Backpressure Signals

Backpressure is expressed via **explicit signaling**, not hidden queues.

Example concepts:

* Demand counters
* Capacity tokens
* Pull-based flow control

No automatic buffering or dropping exists by default.

---

### 6.3 Relationship to CSP

Backpressure integrates naturally with CSP:

* Channel capacity
* Explicit send/receive coordination
* Deterministic flow control

---

## 7. Explicit Exclusions

The extended concurrency API **does not include**:

* Futures / promises
* Async / await
* Virtual threads (duplicate of fibers)
* Reactive frameworks
* Multiple competing abstractions
* Implicit schedulers

If CSP or structured concurrency can solve it, **no new tool is added**.

---

## 8. Relationship to Higher-Level Frameworks

Advanced models such as:

* Actor systems
* Streaming engines
* Dataflow runtimes
* Big-data processing

Belong to:

```
std-framework.*
or
external frameworks
```

They are **not part of std-api.ext.concurrency**.

---

## 9. Stability and Evolution

Rules:

* This API evolves slowly
* New primitives require strong justification
* No overlapping abstractions allowed
* Removal is allowed only with major version bumps

---

## 10. Summary

`std-api.ext.concurrency` provides:

* Atomics for low-level control
* Locks for unavoidable shared mutation
* Cancellation for cooperative shutdown
* Backpressure for controlled flow

It intentionally avoids:

* Over-engineering
* Feature duplication
* Framework-level abstractions

**Core concurrency remains the primary model.
Extended concurrency exists for edge cases only.**

---

**This document is finalized and stable.**

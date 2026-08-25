# Bestie Standard Library — Concurrency

This document defines **`bestie.lib.concurrency`**.

Concurrency helpers sit in **std-lib**, not std-api: they do not talk to files, sockets, HTTP, or the OS process table. They are coordination tools built on the core `thread` primitive (and, for atomics, on CPU instructions).

Import:

```bestie
import bestie.lib.concurrency
```

---

## 1. Design Principles

1. **No duplication of core semantics** — `thread` remains the only parallel execution primitive
2. **No alternative concurrency models** — no async/await, no futures, no virtual threads
3. **Explicit over implicit** — no hidden scheduler on programs that do not import this package
4. **Zero magic** — a fiber scheduler exists only in binaries that use `fiber`
5. **Same ownership rules as `thread`** — `fiber.of` is compiler-known; illegal sharing is a compile-time error

If a problem can be solved with `thread` plus channels or locks, **do not invent a new abstraction**.

---

## 2. What This Package Contains

| Tool | Role |
| ---- | ---- |
| `fiber` | Cooperative lightweight execution on a host `thread` |
| `Channel<T>` | Bounded message passing (CSP send/receive) |
| `atomic<T>` | Explicit atomic operations on primitives |
| `Lock` | Last-resort exclusive lock for shared mutation |
| `CancellationToken` | Cooperative cancellation flag, passed explicitly |

### What this package does **not** contain

* Structured concurrency / supervised task groups
* Actors
* Futures, promises, `async` / `await`
* Reactive streams (see `std-framework/stream.md` if that layer exists)
* A standalone “backpressure” type — bounded `Channel` capacity *is* backpressure
* Deadlock-freedom proofs — locks can deadlock; that is the programmer’s problem
* Parallel collections or work-stealing pools (may be added later if justified)

---

## 3. Fibers (`fiber`)

`fiber` is a **cooperative lightweight thread**. It is the concurrent (not parallel) execution helper.

* Low-overhead, high-concurrency
* Cooperatively scheduled — yields control explicitly
* Runs on an OS `thread` managed by this package’s scheduler
* Does **not** run in parallel with other `fiber` instances on the same scheduler
* Deterministic lifecycle within one scheduler

A program that never calls `fiber.of` does not include the scheduler.

**Factory creation** — same shape as `thread.of`. There is no `.start { }` block form.

```bestie
import bestie.lib.concurrency.fiber

val f = fiber.of(() => compute())
```

**Available methods:**

| Method                | Description                                    |
| --------------------- | ---------------------------------------------- |
| `await()`             | Waits until the fiber completes                |
| `isCompleted(): bool` | Checks if execution finished                   |
| `cancel()`            | Cancels execution (explicit handling required) |
| `id(): int`           | Returns internal fiber identifier              |

**Yield points** — a `fiber` yields at:

* `await()` calls on other fibers
* Explicit `fiber.yield()`
* Blocking I/O calls (the scheduler parks the fiber and resumes when ready)

There is no preemptive scheduling of fibers. Execution is deterministic within a scheduler.

### 3.1 Class Rules

`fiber` is a closed class, not inheritable, not a value class, created **only** via `fiber.of`.

```bestie
fiber.new()       // ❌ forbidden
fiber.of(...)     // ✅
```

### 3.2 Ownership, panic, and bodies

The same rules as core `thread`:

* Implicit `own` sharing into `fiber.of` is a compile-time error
* `move` into the fiber body is allowed
* `ref` cannot cross the spawn boundary
* Immutable values may be shared
* Body returns `void`
* A panic inside a fiber **terminates the entire program** (same as `thread`)

`fiber.of` is **compiler-known**: the frontend applies the same spawn-site ownership analysis as `thread.of`. It is still a library type — it is not a keyword.

### 3.3 Parallel vs concurrent

| Primitive | Layer | Model | Scheduling | Parallelism |
| --------- | ----- | ----- | ---------- | ----------- |
| `thread`  | core | OS thread | Preemptive | Yes — explicit, guaranteed |
| `fiber`   | std-lib | Fiber | Cooperative | No — concurrent only |

Use `thread` when work must run **at the same time** on multiple cores.
Use `fiber` when you need high-concurrency **I/O or coordination** without the cost of OS threads.

### 3.4 `threadlocal`

Fibers share the `threadlocal` copy of the OS thread they run on. See `core/concurrency.md` §7.

---

## 4. Channels (`Channel<T>`)

`Channel<T>` is bounded, explicit message passing between `thread`s and/or `fiber`s.

```bestie
val ch = Channel<int>.of(16)          // capacity 16
val sync = Channel<str>.of(0)         // rendezvous: send waits for receive
```

| Method | Description |
| ------ | ----------- |
| `Channel<T>.of(capacity: int)` | Create; `capacity == 0` is rendezvous |
| `send(value: T): void ! ChannelError` | Blocks if full; error if closed |
| `receive(): T ! ChannelError` | Blocks if empty; error if closed and drained |
| `close(): void` | Further sends fail; remaining values may still be received |

Rules:

* `T` may be a value type, `own` (moved through the channel), or an error union (`int ! WorkError`)
* Sending `own` transfers ownership to the receiver
* `ref` must not be sent — compile-time error (same as crossing a thread boundary)
* Capacity is the backpressure mechanism: a full channel blocks the sender
* No unbounded default; capacity is always explicit
* Closing is idempotent; double-close of an already-closed channel is a no-op

```bestie
errors ChannelError {
    Closed
}
```

---

## 5. Atomic Operations

Atomics exist for **low-level performance-critical** code: lock-free structures, runtime internals. They are not the default tool for application logic.

```bestie
atomic<T>
```

Rules:

* `T` must be a primitive value type
* Operations are explicit
* No implicit memory ordering
* No hidden fences

```bestie
val counter = atomic<int>.of(0)
counter.increment()
```

Atomics are library types, not keywords. They compile to CPU atomic instructions; they do not require `bestie.api.os`.

---

## 6. Locks

Locks exist **only when mutation must be shared** and ownership transfer is impractical. They are a last resort.

```bestie
val lock = Lock.of()
lock.acquire()
updateSharedState()
lock.release()
```

Properties:

* Explicit acquire/release
* No implicit locking
* No reentrancy by default
* Does not bypass `own`/`ref` rules
* Does **not** promise deadlock freedom

Prefer `defer lock.release()` after `acquire()`.

---

## 7. Cancellation

`thread.interrupt()` and `fiber.cancel()` are the primitive cancel signals. `CancellationToken` is the **composable** form for passing a cancel flag through library calls.

```bestie
class CancellationToken {
    fun cancelled(): bool
    fun cancel(): void
}
```

Rules:

* Explicit propagation — pass the token as a parameter
* Cooperative only — the body must poll `cancelled()`
* No forced termination
* No async exceptions

```bestie
fun worker(token: CancellationToken): void {
    while (!token.cancelled()) {
        doWork()
    }
}
```

---

## 8. Relationship to Other Layers

| Layer | Responsibility |
| ----- | -------------- |
| Core `thread` / `threadlocal` | Parallelism and TLS |
| `bestie.lib.concurrency` | Fibers, channels, atomics, locks, cancel tokens |
| `bestie.api.io` / `network` / `http` | Blocking I/O; park a `fiber` or block a `thread` |
| `bestie.framework.*` | Actors, streaming, task groups — if they exist |

---

## 9. Summary

`bestie.lib.concurrency` provides:

* `fiber` for cooperative concurrency
* `Channel<T>` for explicit message passing and backpressure
* `atomic<T>` for lock-free primitives
* `Lock` for unavoidable shared mutation
* `CancellationToken` for cooperative shutdown

It intentionally avoids overlapping concurrency models.

**Core `thread` remains the primary parallel primitive.
This library exists so Bestie does not put a fiber runtime in the language.**

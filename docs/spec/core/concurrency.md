# Bestie Core Concurrency Model

This document defines the **core concurrency primitive** of Bestie.

Core concurrency in Bestie is:

* Low-level
* Deterministic
* Explicit
* Safe-by-default for `own` accounting, with first-class explicit low-level control via `ptr`

**Parallelism is a core feature**, available directly through `thread` — the language's 1:1 OS-thread primitive.

Cooperative fibers, channels, atomics, and locks live in **`bestie.lib.concurrency`**. They are helpers, not language structure. A program that never imports that library has no fiber scheduler and no extra concurrency runtime.

> Unlike Go/Java runtime models, Bestie does not implicitly create an extra `main` worker thread.
> Program entry starts on the process entry OS thread, and `main` executes there with `thread` semantics.

---

## 1. Design Principles

1. Minimal — one execution primitive in core
2. Explicit — no implicit scheduling or sharing
3. Compile-time validated ownership and sharing
4. Deterministic lifecycle
5. Low-overhead

**No implicit sharing, hidden scheduling, or runtime magic.**

---

## 2. OS Threads (`thread`)

`thread` maps 1:1 to an OS-level thread. It is the **parallel execution primitive**.

* Heavyweight, long-lived
* Mapped 1:1 to OS threads
* Owns its stack
* Fully deterministic lifecycle
* Runs in parallel on separate cores — **parallelism is explicit and guaranteed**

**Factory creation:**

```bestie
val t = thread.of(() => work())
```

**Available methods:**

| Method                 | Description                                               |
| ---------------------- | --------------------------------------------------------- |
| `join()`               | Waits until the thread finishes                           |
| `isAlive(): bool`      | Checks if the thread is still running                     |
| `priority(level: int)` | Suggests execution priority (optional, platform-specific) |
| `id(): int`            | Returns thread identifier                                 |
| `interrupt()`          | Signals cancellation — must be handled explicitly in body |

---

## 3. Class Rules

`thread` is:

* A closed class
* Not inheritable
* Not a value class
* Created **only via factory**

```bestie
thread.new()      // ❌ forbidden
thread.init()     // ❌ forbidden
thread.of(...)    // ✅
```

No inheritance, reflection, or dynamic creation is allowed for the core thread type.

---

## 4. Ownership & Sharing

### 4.1 Forbidden

```bestie
val own u = User().new()
thread.of(() => use(u))    // ❌ compile error: implicit ownership sharing
```

### 4.2 Explicit Ownership Transfer

Ownership may be moved into a thread explicitly. The source binding becomes invalid immediately.

```bestie
val own u = User().new()
thread.of(move u, (own user: User) => use(user))   // ✅

use(u)    // ❌ compile error: moved value
```

### 4.3 Allowed — Immutable Sharing

Thread safety comes from the **type**, not the binding. `val` makes a binding immutable — it does not make the value deeply immutable or safe to share.

Safe to share across threads — deep immutability must be explicit:

* Primitive types (`int`, `float64`, `bool`, `char`, etc.)
* `data class`, `value class`, `enum` — value semantics by definition
* Classes annotated `@immutable`
* `const` values
* Immutable collections — built with `.immutable`: `list<T>.immutable.build()`

```bestie
val cfg = Config.load()                          // Config is a data class ✅
val xs  = list<int>.immutable.build()            // explicitly immutable ✅
thread.of(() => read(cfg))                       // ✅ safe
thread.of(() => process(xs))                     // ✅ safe
```

**Not** safe to share:

```bestie
val xs: list<int> = list<int>.build()
thread.of(() => read(xs))    // ❌ compile error: mutable list, val binding is not enough
```

---

## 5. Compile-Time Guarantees

* Implicit ownership sharing across threads is a **compile-time error**
* `own` cannot be implicitly shared; transfer with `move`
* `ptr<T>` may cross thread boundaries; races and lifetime are programmer-owned
* A `ref` field is not a thread-safe handle — sharing the object it names is done with `ptr<T>`

> If `own` sharing rules compile, ownership accounting is safe by construction.
> `ptr`-based sharing remains explicit programmer responsibility.

The same ownership rules apply to compiler-known spawn points in `bestie.lib.concurrency` (`fiber.of`). See that package.

---

## 6. Panic Behavior in Threads

A panic represents a violated invariant — the program is in an invalid state. There is no partial recovery.

**A panic in any thread terminates the entire program.**

There is no mechanism to catch or isolate a thread panic at the core level.

```bestie
val t = thread.of(() => {
    panic("something is wrong")   // entire program terminates
})
t.join()
```

### Thread Bodies Return `void`

Core thread bodies return `void`. There is no return value from a thread at the core level.

```bestie
thread.of(() => doWork())       // void body
```

### Communicating Results and Errors

To get results or typed errors (`!`) back from a worker thread, use **channels** from `bestie.lib.concurrency`:

```bestie
import bestie.lib.concurrency.Channel

val ch = Channel<int ! WorkError>.of(1)
thread.of(move ch, (ch) => {
    val outcome = try compute()
    ch.send(outcome)
})
val outcome = try ch.receive()
```

This keeps the core minimal and the result-passing pattern explicit.

---

## 7. Thread-Local Storage

`threadlocal` is a **storage modifier** — not a class, not a wrapper, not a container. It lives with the concurrency model because it is meaningful only in the presence of `thread`.

A `threadlocal` declaration gives each OS thread its **own independent copy** of the variable. Access is direct, with no `.get()`, `.set()`, or any other wrapper indirection.

```bestie
threadlocal val requestId: str = ""
threadlocal var callDepth: int = 0
```

### 7.1 Usage

Access and mutation are identical to any other variable:

```bestie
threadlocal var counter: int = 0

fun tick() {
    counter += 1
}
```

There is no ceremony. `counter` is a variable. The compiler routes reads and writes to the calling thread's copy.

### 7.2 Semantics

* Each `thread` gets its **own independent copy**, initialized from the declared initializer
* Fibers from `bestie.lib.concurrency` share the copy of the OS thread they run on
* Initialization is **per-thread at first access** (lazy, but compile-time lowered to a TLS slot — no heap allocation)
* The initializer expression must be a **compile-time constant** or a pure function of compile-time constants
* Copies are independent — writes in one thread are invisible to others

### 7.3 Scope

`threadlocal` is allowed at:

* **Module level** (top-level declaration)
* **Function level** (static local — initialized once per thread, not once per call)

`threadlocal` inside a regular expression or block (non-static context) is a **compile-time error**.

### 7.4 Rules

* `threadlocal val` — immutable per-thread binding (the value is constant for that thread's lifetime)
* `threadlocal var` — mutable per-thread binding
* The initializer must be compile-time constant
* No sharing across threads — compiler enforces this for `own` values
* `ptr` into a `threadlocal` from another thread is legal but programmer-responsibility (same as all `ptr` use)
* No runtime overhead beyond the platform TLS mechanism (hardware register + offset on x86_64/arm64)
* No `@threadlocal` annotation, no `ThreadLocal<T>` class — `threadlocal` is a first-class storage modifier

---

## 8. What Core Does Not Include

Core provides `thread` and `threadlocal`. Everything else is **`bestie.lib.concurrency`**, built on top:

* Fibers (`fiber`)
* Channels
* Mutexes, atomics
* Cooperative cancellation tokens
* Thread pools and work-stealing schedulers

These are **library helpers** — not language features and not OS APIs.

Actors, structured concurrency, and reactive pipelines belong in frameworks or application code, not in core and not in this library's required surface.

---

## 9. Summary

| Need | Use |
| ---- | --- |
| Parallel execution on multiple cores | `thread` (core) |
| High-concurrency cooperative I/O | `fiber` (`bestie.lib.concurrency`) |
| Message passing between threads | `Channel` (`bestie.lib.concurrency`) |
| Shared mutable state | Locks / atomics (`bestie.lib.concurrency`) |

> `thread` is the only core execution primitive.
> Parallelism is in core. Fibers and coordination are in std-lib.

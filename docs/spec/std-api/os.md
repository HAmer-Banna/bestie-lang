# bestie.api.os — Operating System API

This document defines the **Bestie Standard OS API (`bestie.api.os`)**.

`bestie.api.os` is how a program talks to the operating system it is running on: processes, environment, signals, clocks, limits, and entropy.

It is **not** a framework, **not** a runtime, and **not** a replacement for the core language.

---

## 1. Scope and Non-Goals

### 1.1 What `bestie.api.os` Provides

* Process creation, waiting, and termination, with pipes as ordinary streams
* Termination of the calling process
* Environment variables and working directory
* Signal handling
* OS clocks — wall time and monotonic time
* Resource limits and thread scheduling
* Runtime platform introspection
* Cryptographically secure entropy

### 1.2 What `bestie.api.os` Does *Not* Provide

* File system operations (see `bestie.api.fs`)
* Networking (see `bestie.api.network`)
* Console I/O (see `bestie.api.io`)
* A shell, or any command-string parsing
* Compile-time platform facts — those are core `target.*` constants (§10)
* Pseudo-random numbers (see `bestie.lib.random`)

---

## 2. Design Principles

1. **Every OS call can fail, and says so.** There are no silent no-ops.
2. **Classes for owned OS resources, value types for descriptions**
3. **Functions for stateless queries**
4. **No global mutable abstractions** — no ambient environment object, no global clock
5. **Reuse the layers below** — process pipes are `bestie.api.io` streams, times are `bestie.lib.datetime` values
6. **Intent is not guarantee.** Where the OS may refuse, the API says so rather than pretending.

---

## 3. Namespacing

```text
bestie.api.os
```

No re-exports.

---

## 4. Class Kinds and Ownership Rationale

| Type | Kind | Why |
| ---- | ---- | --- |
| `Process` | `class` | Owns a child process handle and its pipe streams. Identity is essential — waiting on "a" process rather than *this* one would be a correctness bug. |
| `SpawnOptions` | `class` | Owns an optional environment map, and is mutated while being configured. |
| `Stdio` | `enum` | A closed set of three dispositions for a child stream. |
| `ExitStatus` | `enum` | A process ends by exiting with a code **or** by being killed by a signal. Two variants, never both. |
| `Signal` | `enum` | A closed set. |
| `Priority` | `enum` | A closed set of scheduling intents. |
| `ResourceKind` | `enum` | A closed set of limit categories. |
| `ResourceLimit` | `value class` | A soft/hard pair. No identity, copied inline. |
| `PlatformInfo` | `data class` | A snapshot of runtime facts, read once and never mutated. Structural equality is meaningful. |

`Process` is the only owned resource here. Everything else is either a value description or a stateless function.

---

## 5. Processes

### 5.1 `Stdio` and `SpawnOptions`

```bestie
enum Stdio {
    Inherit,     // child shares this process's stream
    Pipe,        // child gets a pipe the parent can read or write
    Null         // child's stream is discarded
}

class SpawnOptions {
    var stdin:  Stdio
    var stdout: Stdio
    var stderr: Stdio
    var workingDir: Path ?          // absent means inherit
    val own env: map<str, str> ?    // absent means inherit the parent's environment
}
```

An absent `env` **inherits**; a present one **replaces** entirely. There is no merge mode, because "merge" needs a collision rule that differs per use.

### 5.2 `Process`

```bestie
class Process {
    fun pid(): int

    val own stdin:  OutputStream ?   // present only when Stdio.Pipe was requested
    val own stdout: InputStream ?
    val own stderr: InputStream ?

    fun wait(): ExitStatus ! OsError
    fun tryWait(): ExitStatus ? ! OsError    // absent when still running
    fun kill(sig: Signal): void ! OsError
    fun free(): void
}

fun spawn(path: str, args: slice<str>, opts: ptr<SpawnOptions>): own Process ! OsError
```

**Pipes are `bestie.api.io` streams.** A child's output is an `InputStream`, so everything that already reads a stream reads a subprocess — `BufferedReader`, `TextReader`, `copy` — with no process-specific code.

`tryWait()` returns `ExitStatus ? ! OsError`: exited with this status, still running, or the query failed. Three outcomes, three-outcome type (`core/types.md` §8.4).

Rules:

* No shell expansion. `path` is an executable, not a command line, and `args` are passed to the child as written — there is no string a shell could reinterpret.
* No implicit `PATH` resolution. Pass an absolute path, or resolve it yourself.
* A `Process` is never implicitly detached or reaped. It is an `own` value: `wait()` then `free()`, or `free()` alone to detach.

### 5.3 `ExitStatus`

```bestie
enum ExitStatus {
    Exited(int32),      // the process returned this code
    Signaled(Signal)    // the process was terminated by this signal
}
```

A process that was killed did not "exit with 137". Collapsing both outcomes into one integer — the shell convention of `128 + signal` — makes a genuine exit code of `137` indistinguishable from a `SIGKILL`. The enum keeps them distinct, and `switch` forces the caller to consider both.

### 5.4 Example — capturing a child's output

```bestie
import bestie.api.io.TextReader
import bestie.api.io.Encoding

fun gitRevision(): str ! (OsError | ReadError) {
    val own opts = SpawnOptions.new(
        stdin:  Stdio.Null,
        stdout: Stdio.Pipe,
        stderr: Stdio.Null,
        workingDir: absent,
        env: absent
    )
    defer opts.freeDeep()

    var args : array<str>[2]
    args.add("rev-parse")
    args.add("HEAD")

    val own proc = try spawn("/usr/bin/git", args[..], opts.address())
    defer proc.free()

    val own out = proc.stdout else { return OsError.NoPipe }
    val own text = TextReader.new(move out, Encoding.Utf8)
    defer text.close()

    val rev = try text.readAll()

    switch (try proc.wait()) {
        case ExitStatus.Exited(val code) if code == 0 => return rev
        case _                                        => return OsError.ChildFailed
    }
}
```

### 5.5 Terminating This Process

```bestie
fun exit(code: int32): never
```

`exit` terminates the calling process immediately. Its return type is `never` (`core/exceptions.md` §4.4), so a call satisfies any expected type and the compiler knows control does not continue past it:

```bestie
val own cfg = loadConfig(path) catch |e| {
    println("config: ${renderError(e)}")
    exit(2)
}
```

Rules:

* `exit` does **not** run `defer` statements, does not call `deinit()`, and does not flush buffered writers. It is process termination, not scope exit — anything that must be flushed or released must be handled first.
* Code `0` conventionally means success. This API assigns no meaning to any other value.
* Returning normally from `main` exits with `0`. `exit` exists for paths that must report a different code, which `main`'s `void` return cannot express.

---

## 6. Environment and Working Directory

```bestie
fun getEnv(key: str): own str ?
fun setEnv(key: str, value: str): void ! OsError
fun removeEnv(key: str): void ! OsError
fun envVars(): own map<str, str> ! OsError

fun currentDir(): own Path ! OsError
fun setCurrentDir(dir: ptr<Path>): void ! OsError
```

There is no ambient environment object. `envVars()` returns a **snapshot** — a copy taken at the call, not a live view, because the environment can change underneath a live view and a snapshot is what callers actually reason about.

The process environment is **process-global mutable state**, which Bestie otherwise avoids. `setEnv` is therefore not thread-safe with respect to a concurrent `getEnv` on most platforms; set what you need before spawning threads.

---

## 7. Signals

### 7.1 `Signal`

```bestie
enum Signal {
    Term, Int, Hup, Quit, Usr1, Usr2, Pipe, Alarm, Child, Kill, Stop
}
```

`Kill` and `Stop` may be **sent** but never handled — the OS does not deliver them to a handler. Registering one is `OsError.Unsupported`.

### 7.2 Handling

```bestie
fun onSignal(sig: Signal, handler: fn(Signal) -> void): void ! OsError
fun resetSignal(sig: Signal): void ! OsError
```

**A signal handler runs in a severely restricted context.** It may interrupt the program at any instruction, including partway through an allocation or while a lock is held. Inside a handler:

* Only **non-capturing** lambdas are accepted (`core/fp.md` §7.2) — a capturing `[x]` lambda is rejected, because a capture struct's lifetime is not guaranteed to outlive an asynchronous delivery
* Do not allocate, do not free, do not take a `Lock`, and do not perform I/O
* Do not `panic` — a panic inside a handler terminates the process from an undefined program point

The safe pattern is to record and return, then act in ordinary code:

```bestie
import bestie.lib.concurrency.atomic
import bestie.lib.concurrency.Ordering

val SHUTDOWN = atomic<bool>.new(false)

fun installShutdown(): void ! OsError {
    try onSignal(Signal.Term, (s: Signal) => SHUTDOWN.store(true, Ordering.Release))
    try onSignal(Signal.Int,  (s: Signal) => SHUTDOWN.store(true, Ordering.Release))
}

fun serveLoop(): void ! OsError {
    while (not SHUTDOWN.load(Ordering.Acquire)) {
        serveOne()
    }
    drainAndClose()
}
```

An atomic store is one of the few operations that is safe from a handler, which is why it is the idiom rather than merely a convention.

---

## 8. Clocks

```bestie
fun now(): Instant ! OsError
fun monotonic(): Duration
```

* `now()` returns an `Instant` from `bestie.lib.datetime` — this package defines no time type of its own. It is wall-clock time and may jump backwards when the system clock is adjusted.
* `monotonic()` returns a `Duration` since an **unspecified fixed point** that does not change while the process runs. It never goes backwards. Its absolute value is meaningless; only differences are.

Measuring an interval always uses `monotonic()`:

```bestie
val start = monotonic()
doWork()
val elapsed = monotonic() - start
```

`monotonic()` is infallible — a platform without a monotonic clock cannot host Bestie.

Time zones are not here. `Instant` is an absolute point; converting it to a local calendar time is `bestie.lib.datetime`.

---

## 9. Resource Limits and Thread Scheduling

### 9.1 Limits

```bestie
enum ResourceKind {
    OpenFiles, StackSize, AddressSpace, CpuTime, CoreDumpSize
}

value class ResourceLimit {
    soft: uint64          // the enforced limit
    hard: uint64          // the ceiling the soft limit may be raised to
}

fun getLimit(kind: ResourceKind): ResourceLimit ! OsError
fun setLimit(kind: ResourceKind, limit: ResourceLimit): void ! OsError
```

Raising a soft limit above the hard limit, or raising the hard limit without privilege, is `OsError.PermissionDenied`.

### 9.2 Thread Scheduling

Core `thread` (`core/concurrency.md` §2) provides spawn, `join`, `isAlive`, `id`, and `interrupt` — the portable surface that means the same thing everywhere. **Scheduling policy is platform-specific and lives here**, because this is the layer allowed to change as operating systems do.

```bestie
enum Priority {
    Idle, Low, Normal, High, Realtime
}

fun setPriority(t: ptr<thread>, level: Priority): void ! OsError
fun getPriority(t: ptr<thread>): Priority ! OsError
fun setAffinity(t: ptr<thread>, cpus: ptr<set<int>>): void ! OsError
fun setName(t: ptr<thread>, name: str): void ! OsError
```

Rules:

* Every call is fallible — an OS may refuse a priority, ignore affinity, or cap a name's length. Failures are `! OsError`, never silent no-ops.
* `Priority` is an **intent**, not a guarantee. The mapping to OS values is platform-defined and documented per target.
* `Realtime` typically requires elevated privileges and fails with `OsError.PermissionDenied` otherwise.
* Affinity and naming are unavailable on some targets; those return `OsError.Unsupported` rather than pretending to succeed.

None of this changes core `thread` semantics — ownership, the spawn-boundary rules, and panic behavior are unaffected.

---

## 10. Platform Introspection

```bestie
data class PlatformInfo {
    osVersion:   str          // e.g. "6.8.0-generic", "14.4"
    hostname:    str
    cpuCount:    int
    pageSize:    int
    totalMemory: uint64
}

fun platform(): own PlatformInfo ! OsError
```

**This is only for facts unknown until run time.** OS family, architecture, word size, and endianness are **compile-time constants** — `target.os`, `target.arch`, `target.bits`, `target.endian` (`core/constants.md` §1) — and branching on them belongs in `when`, where the losing branch is erased from the binary:

```bestie
when (target.os == "linux") {
    useEpoll()
} else {
    usePoll()
}
```

Querying at run time what the compiler already knows would cost a branch, keep dead code in the binary, and invite the two facts to disagree. A runtime query is for what genuinely varies between machines running the same binary: how many CPUs this one has, how much memory, what it is called.

---

## 11. Entropy and Secure Randomness

The operating system is the **only** source of true entropy. Pseudo-random generation lives in [`bestie.lib.random`](../std-lib/random.md); this section provides the raw, cryptographically secure material PRNGs may be seeded from and that security-sensitive code must use directly.

```bestie
fun entropy64(): uint64 ! EntropyError
fun secureBytes(dst: slice<var byte>): void ! EntropyError
```

Rules:

* Output is **cryptographically secure** — suitable for keys, tokens, and nonces
* Each call draws fresh entropy; there is no internal cached state
* `secureBytes` fills a caller-owned view in place — no allocation, and the buffer's lifetime is the caller's
* Blocking vs non-blocking behavior is platform-defined but never silently degraded

```bestie
errors EntropyError {
    Unavailable,    // no OS entropy source accessible
    WouldBlock      // non-blocking source not yet seeded
}
```

Entropy access is fallible because the source may genuinely be unavailable — early boot, a sandbox, an exhausted handle.

| Package | Guarantee |
| ------- | --------- |
| `bestie.api.os` | True entropy, fallible, OS-backed, **secure** |
| `bestie.lib.random` | Deterministic PRNG, reproducible, fast, **not secure** |

---

## 12. Error Model

```bestie
errors OsError {
    NotFound,           // executable, variable, or path does not exist
    PermissionDenied,
    Unsupported,        // the platform does not offer this operation
    Interrupted,
    ResourceExhausted,  // out of processes, descriptors, or memory
    InvalidArgument,
    NoPipe,             // a pipe stream was requested that was not configured
    ChildFailed,        // a spawned process exited non-zero or was signaled
    Timeout
}
```

Rules:

* No exceptions, no implicit retries. An `Interrupted` call is returned to you; whether to retry is your policy.
* An operation a platform does not support returns `Unsupported`. It never succeeds silently and never approximates.
* Errors from lower layers keep their own sets: pipe reads fail with `ReadError`, entropy with `EntropyError`. They compose per `core/exceptions.md` §3.1.

---

## 13. Relationship to Other Layers

| Concern | Package |
| ------- | ------- |
| Process pipe streams | `bestie.api.io` |
| Paths | `bestie.api.fs` |
| `Instant` and `Duration` | `bestie.lib.datetime` |
| Atomics used in signal handlers | `bestie.lib.concurrency` |
| Compile-time platform facts | core `target.*` (`core/constants.md`) |
| Seeded PRNGs | `bestie.lib.random` |

---

## 14. Stability and Evolution

`bestie.api.os` is the layer **most expected to change**, because operating systems do. Platform-specific additions live under `bestie.api.os.linux`, `bestie.api.os.windows`, and so on, and may not alter the semantics defined here.

---

## 15. Summary

`bestie.api.os` gives a program deterministic access to the system underneath it:

* **Every call is fallible and says so.** Nothing silently no-ops when a platform cannot comply.
* **Process pipes are streams.** A subprocess's output is an `InputStream`, so every stream tool reads it with no process-specific code.
* **`ExitStatus` keeps exit and signal distinct**, instead of the `128 + signal` convention that makes a genuine code of `137` ambiguous.
* **Signal handlers record; ordinary code acts.** Non-capturing only, no allocation, no locks — the atomic-flag pattern is the idiom because it is one of the few safe operations.
* **Runtime introspection is only for what varies at run time.** OS, architecture, and endianness are compile-time `target.*` constants resolved by `when`, not runtime queries.
* **Time comes from `bestie.lib.datetime`**, and intervals are always measured with `monotonic()`.

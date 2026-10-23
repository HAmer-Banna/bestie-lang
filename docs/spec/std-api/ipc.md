# bestie.api.ipc — Interprocess Communication API

This document defines the **Bestie Standard IPC API (`bestie.api.ipc`)**.

`bestie.api.ipc` is how cooperating processes on **the same machine** exchange bytes, share memory, and coordinate: pipes, local sockets, shared memory, and named semaphores.

It is designed for:

* Supervisors and their workers
* Daemons and the command-line tools that drive them
* Multi-process servers sharing a cache or a ring buffer
* Privilege separation — a small trusted process and a large untrusted one

It is **not** an RPC framework, **not** a message broker, and **not** a serialization format.

---

## 1. Scope and Non-Goals

### 1.1 What `bestie.api.ipc` Provides

* Anonymous pipes, implementing the stream protocols of `bestie.api.io`
* Local (Unix-domain) stream sockets, implementing the same protocols
* Named and anonymous shared memory, with typed access checked for cross-process safety
* Named counting semaphores
* Handing any of the above to a child process explicitly, by token

### 1.2 What `bestie.api.ipc` Does *Not* Provide

* Child stdio pipes — those are `Stdio.Pipe` in `bestie.api.os` (§5 there), and remain the simplest channel to a child
* Signals (see `bestie.api.os` §7)
* Communication between machines (see `bestie.api.network`)
* Serialization or message framing (see `bestie.lib.format`)
* RPC, service discovery, or brokers
* Desktop buses and object systems — D-Bus, COM, XPC, Binder
* System V IPC (`shmget`, `semget`, `msgget`) — superseded on every target by the mechanisms here
* Memory-mapped files
* Anything a portable target cannot honor with the same meaning — those live in platform extensions (§13)

---

## 2. Design Principles

1. **Every mechanism here exists on every hosted target with the same meaning.** If one OS cannot give it the semantics stated here, it is not in this package — it is in an extension.
2. **Byte channels are streams.** A pipe and a local socket are `InputStream` / `OutputStream`, so every stream tool works over them unchanged.
3. **Handles never leak into children.** Every handle this package creates is non-inheritable; passing one to a child is a visible line of code.
4. **Shared memory is typed for the boundary it crosses.** Only types whose bytes mean the same thing in another process may be placed in it — checked at compile time.
5. **Named objects outlive programs, so removal is explicit.** Nothing is unlinked on your behalf.
6. **No hidden synchronization.** Shared memory is ordinary memory; ordering is written with atomics, exactly as between threads.
7. **Intent is not guarantee.** Where an OS may refuse — a timed wait, a permission — the API says so rather than emulating.

---

## 3. Namespacing

```text
bestie.api.ipc
```

No re-exports. `HandleToken` is defined in `bestie.api.os` (§5.1 there), because it belongs to process spawning; this package imports it.

---

## 4. Class Kinds and Ownership Rationale

| Type | Kind | Why |
| ---- | ---- | --- |
| `PipeReader` | `class` | Owns the read end of a pipe. Identity matters: closing *this* end is what delivers end-of-stream to the writer's peer. |
| `PipeWriter` | `class` | Owns the write end. Same. |
| `Pipe` | `class` | Owns both ends until they are moved out, the way `Process` owns its stdio streams. |
| `LocalSocket` | `class` | A connected local socket — a live descriptor with identity. |
| `LocalListener` | `class` | Owns a bound, listening descriptor. |
| `SharedMemory` | `class` | Owns a mapping and the handle behind it. Must not be aliased within the process, and `own` enforces that. |
| `Semaphore` | `class` | Owns a handle to a named kernel object. |
| `IpcName` | `class` | A validated, portable name for a shared-memory object or semaphore. A `class` only because its `init` is fallible; it holds no handle, and two equal names designate the same object. |
| `Audience` | `enum` | A closed set: who besides the creator may open a named object. |
| `Access` | `enum` | A closed set: read-only or read-write mapping. |
| `LocalOptions` | `value class` | A settings bundle. Copied by value. |

Every class here owns an OS handle, so every one is an `own` value with an explicit `close()` (`core/memory.md` §7), and none is copyable — a copied handle is a double close.

---

## 5. Naming

### 5.1 `IpcName`

Shared memory and semaphores are found by name. Operating systems disagree about what a name may be — POSIX wants a leading `/` and no other slash, macOS caps the length at 31 bytes, Windows reserves `\` and a `Global\` prefix — so Bestie defines the intersection and maps it:

```bestie
class IpcName {
    init(text: str): ! IpcError
    fun toStr(): str
}
```

Rules:

* `text` is 1 to 30 bytes from `[A-Za-z0-9._-]`. Anything else is `IpcError.InvalidName` at construction, on every target — a name that works on Linux and fails on macOS is rejected on Linux too.
* The platform prefix is added by the implementation: `/name` on POSIX, `Local\name` on Windows. The caller never writes it.
* Names are per-machine, per-user-session on Windows (`Local\`). Reaching the global namespace is privileged and lives in `bestie.api.ipc.windows`.

`IpcName` is a `class` rather than a `value class` only because its `init` is fallible (`core/oop.md` §11.6). It holds no handle and needs no `close()`.

### 5.2 Local Socket Paths

Local sockets are addressed by a `Path` from `bestie.api.fs`, because on every target — including Windows' `AF_UNIX` — a local socket is a file-system entry. The OS limit on that path is about 104 bytes (`sun_path`); a longer one is `IpcError.NameTooLong`, never a silent truncation onto a different path.

### 5.3 `Audience`

```bestie
enum Audience {
    CurrentUser,     // only processes running as the creating user
    AnyUser          // any local process
}
```

`Audience` is an intent mapped to the platform's access control — mode bits on POSIX, a security descriptor on Windows. Finer control (groups, ACL entries) is platform-specific and lives in the extensions.

---

## 6. Pipes

A pipe is a one-way, unnamed byte channel. It is the right tool when one process creates the channel and hands one end to a process it spawns.

```bestie
import bestie.api.io.InputStream
import bestie.api.io.OutputStream

class PipeReader impl InputStream {
    fun read(dst: slice<var byte>): int ! ReadError
    fun token(): HandleToken
    fun close(): void
}

class PipeWriter impl OutputStream {
    fun write(src: slice<byte>): int ! WriteError
    fun flush(): void ! WriteError
    fun token(): HandleToken
    fun close(): void
}

class Pipe {
    val own reader: PipeReader ?
    val own writer: PipeWriter ?
    fun free(): void
}

fun pipe(): own Pipe ! IpcError
fun atomicWriteLimit(): int
```

Rules:

* A `read` of `0` means every writer has closed — end of stream, not an error (`io.md` §5). This is also the portable way to learn that the peer process has **died**: the OS closes its handles.
* A write to a pipe whose reader is gone fails with `WriteError.Closed`. It does not kill the process: the hosted runtime sets `Signal.Pipe` to ignored before `main` (`os.md` §7).
* A single `write` of at most `atomicWriteLimit()` bytes is never interleaved with another writer's. Larger writes may be. The limit is `PIPE_BUF` on POSIX (at least 512); it is `0` on targets that make no such promise.
* No buffering beyond the kernel's. Wrap in `BufferedWriter` when you want it.

---

## 7. Local Sockets

A local socket is a **bidirectional, connection-oriented** byte stream between processes on one machine, with a listener at a known path. It is the right tool when the processes did not start each other — a daemon and its clients.

```bestie
import bestie.api.network.Shutdown

class LocalSocket impl InputStream, OutputStream {
    fun read(dst: slice<var byte>): int ! ReadError
    fun write(src: slice<byte>): int ! WriteError
    fun flush(): void ! WriteError

    fun peerPid(): int ! IpcError
    fun setOptions(opts: LocalOptions): void ! IpcError
    fun shutdown(how: Shutdown): void ! IpcError
    fun token(): HandleToken
    fun close(): void
}

class LocalListener {
    fun accept(): own LocalSocket ! IpcError
    fun path(): Path
    fun close(): void
}

value class LocalOptions {
    readTimeout:  Duration ?     // absent means block indefinitely
    writeTimeout: Duration ?
}

fun listenLocal(path: Path, backlog: int, audience: Audience): own LocalListener ! IpcError
fun connectLocal(path: Path): own LocalSocket ! IpcError
fun connectLocal(path: Path, timeout: Duration): own LocalSocket ! IpcError
```

`Shutdown` is reused from `bestie.api.network` — a half-close means the same thing on a local socket, and one vocabulary serves both.

Rules:

* `peerPid()` reports the process at the other end, as the **kernel** recorded it at connect time — not a value the peer claimed. It is the basis for authorizing a client. A target that cannot report it returns `IpcError.Unsupported`.
* `listenLocal` fails with `IpcError.AddressInUse` if anything exists at `path`. **`close()` does not remove the socket file.** A stale file left by a crashed server is the caller's to detect and remove (§7.1) — removing a path is a file-system operation, done with `bestie.api.fs.remove`, never implicitly.
* A write to a socket whose peer is gone fails with `WriteError.Closed`, never `Signal.Pipe`.
* Stream sockets only. Datagram and sequenced-packet local sockets, and passing descriptors over a socket, are not available on Windows and live in `bestie.api.ipc.unix` (§13).

### 7.1 Example — a daemon socket with stale-file recovery

```bestie
import bestie.api.fs.Path
import bestie.api.fs.remove

fun bindControl(path: Path): own LocalListener ! (IpcError | FsError) {
    val own first = listenLocal(path, backlog: 16, Audience.CurrentUser) catch |e| {
        if (e != IpcError.AddressInUse) { return e }

        // Something is there. If a live server answers, it is not ours to take.
        val own probe = connectLocal(path) catch |c| {
            if (c != IpcError.Refused) { return c }
            try remove(path)                 // nobody listening: a stale file
            return try listenLocal(path, backlog: 16, Audience.CurrentUser)
        }
        probe.close()
        return IpcError.AddressInUse
    }
    return first
}
```

The probe-then-remove has a window in which a second server could start; two servers racing for one path is resolved by the loser's `AddressInUse`, not by this function. A program that needs a stronger guarantee holds a lock file alongside the socket.

---

## 8. Shared Memory

### 8.1 `SharedMemory`

```bestie
enum Access {
    ReadOnly,
    ReadWrite
}

class SharedMemory {
    fun size(): uint
    fun name(): IpcName ?                              // absent for anonymous memory

    fun at<T>(offset: uint): ptr<const T> ! IpcError
    fun atMut<T>(offset: uint): ptr<T> ! IpcError

    fun read(offset: uint, dst: slice<var byte>): void ! IpcError
    fun write(offset: uint, src: slice<byte>): void ! IpcError

    fun token(): HandleToken
    fun close(): void
}

fun createShared(name: IpcName, size: uint, audience: Audience): own SharedMemory ! IpcError
fun openShared(name: IpcName, access: Access): own SharedMemory ! IpcError
fun anonymousShared(size: uint): own SharedMemory ! IpcError
fun removeShared(name: IpcName): void ! IpcError
```

Rules:

* `createShared` is **exclusive**: it fails with `IpcError.AlreadyExists` if the name exists. There is no create-or-open mode, because "open whatever is there" silently adopts a stale object with a stale size and stale contents. A caller that wants that behavior writes the fallback itself.
* New memory is zero-filled. `createShared` maps it `ReadWrite`.
* The size is fixed at creation. Resizing is not portable (Windows cannot) and would invalidate every pointer into the mapping in every process.
* `close()` unmaps and releases the handle. It does **not** remove the name.
* `anonymousShared` has no name. It is reachable from another process only by passing its `token()` to a child (§10).

### 8.2 What May Live in Shared Memory

`at<T>` and `atMut<T>` accept a `T` only if its bytes mean the same thing in another process. The compiler checks this:

| Permitted | Why |
| --------- | --- |
| Fixed-width integers, `float32`, `float64`, `byte`, `bool` | Size and representation are fixed by the target ABI |
| `atomic<P>` of a permitted primitive, where lock-free on the target | Cross-process atomics must be the hardware instruction; a lock-based fallback would lock a lock only one process can see |
| A `@repr(C)` type whose fields are all permitted | Declaration-order layout, fixed by the C ABI (`foreign.md` §5.2) |
| `array<T>[n]` of a permitted `T` | Inline, fixed stride |

| Rejected | Why |
| -------- | --- |
| `int`, `uint` | Target-width — a 32-bit and a 64-bit process would disagree |
| Any non-`@repr(C)` aggregate | Its layout is not stable across builds (`memory-layout.md` §15) |
| `ptr<T>`, `str`, `slice<T>`, any `own` or `ref` field, any `class` | They hold addresses, and an address is meaningful in exactly one process |

Using a rejected `T` is a compile-time error naming the offending field. `atomic<P>` on a target where it is not lock-free is also a compile-time error, on that target.

Additionally, at run time:

* `offset` must satisfy `alignOf(T)`, or the call fails with `IpcError.Misaligned`.
* `offset + sizeOf(T)` must not exceed `size()`, or the call fails with `IpcError.OutOfRange`. Unlike an `MmioRegion` index (`memory.md` §5), this is an **error, not a panic**: the size of an opened object was chosen by another process, so an offset that does not fit is an input the program must handle, not a bug it can rule out.
* `atMut<T>` on a `ReadOnly` mapping fails with `IpcError.PermissionDenied`.

### 8.3 Synchronization

Shared memory is **ordinary memory that another program also writes**. The rules are the rules for threads (`core/concurrency.md` §8):

* Concurrent access where at least one side writes must go through `atomic<P>`, or be ordered by an atomic release/acquire pair, or by a `Semaphore` (§9).
* A plain read through a `ptr<T>` of a location another process is writing concurrently is a data race, with the same consequences as between threads — the optimizer may keep the value in a register.
* There are no implicit fences. Every atomic operation names its `Ordering`.

`SharedMemory` hands out `ptr<T>`, never `slice<var T>`. A mutable slice is **exclusive** within the process (`core/types.md`), and the compiler is entitled to optimize on that — but no compiler can see the other process. A `ptr<T>` carries no such promise. `read` and `write` copy whole ranges and are always safe, because each is one opaque operation the optimizer cannot split or cache.

### 8.4 Example — a single-producer, single-consumer counter page

```bestie
import bestie.lib.concurrency.atomic
import bestie.lib.concurrency.Ordering

@repr(C)
value class Stats {
    requests: atomic<uint64>
    errors:   atomic<uint64>
}

fun publishStats(name: IpcName): own SharedMemory ! IpcError {
    val own mem = try createShared(name, sizeOf(Stats), Audience.CurrentUser)
    return mem
}

fun countRequest(mem: ptr<SharedMemory>): void ! IpcError {
    val s = try mem.val.atMut<Stats>(0)
    s.val.requests.add(1, Ordering.Relaxed)
}

// In a monitoring process:
fun readRequests(name: IpcName): uint64 ! IpcError {
    val own mem = try openShared(name, Access.ReadOnly)
    defer mem.close()
    val s = try mem.at<Stats>(0)
    return s.val.requests.load(Ordering.Relaxed)
}
```

Removing `@repr(C)` from `Stats` makes both `atMut<Stats>` and `at<Stats>` compile errors: the monitor may be a different build, and the two would disagree on where `errors` lives.

### 8.5 Lifetime of Named Objects

Operating systems disagree on when a named object dies. On POSIX it persists until `removeShared` or reboot, even with no process using it. On Windows it is destroyed when the last handle closes.

Portable code therefore follows one rule: **the creator calls `removeShared` when the object is no longer needed, and no process relies on the object surviving after every handle is closed.** A program that does both is correct on every target; a program that does neither leaks on POSIX.

On POSIX, removing a name while other processes have it mapped is safe — their mappings remain valid, and the name is free for a new object.

---

## 9. Semaphores

A named counting semaphore coordinates processes that share no other channel, or guards a region of shared memory too large for atomics.

```bestie
class Semaphore {
    fun wait(): void ! IpcError
    fun wait(timeout: Duration): void ! IpcError
    fun tryWait(): bool ! IpcError        // false when the count is zero
    fun post(): void ! IpcError
    fun close(): void
}

fun createSemaphore(name: IpcName, initial: uint32, audience: Audience): own Semaphore ! IpcError
fun openSemaphore(name: IpcName): own Semaphore ! IpcError
fun removeSemaphore(name: IpcName): void ! IpcError
```

Rules:

* `createSemaphore` is exclusive, like `createShared`, and `close()` does not remove the name (§8.5 applies unchanged).
* A timed `wait` that expires fails with `IpcError.TimedOut`. On a target with no native timed wait on a named semaphore — macOS is one — it fails with `IpcError.Unsupported`. It is never emulated with a polling loop, because a poll interval is latency and CPU the caller did not choose.
* `post` beyond the platform maximum is `IpcError.ResourceExhausted`.
* A binary semaphore (`initial: 1`) is the portable cross-process lock.

**A semaphore does not know who holds it.** A process that dies between `wait` and `post` leaves the count decremented forever, and no portable mechanism can detect it. Where owner death must be recoverable, either:

* tie the resource to a pipe or local socket, whose end-of-stream **is** delivered when the holder dies (§6), or
* use a robust process-shared mutex from `bestie.api.ipc.linux` (§13), on the targets that have one.

**A semaphore wait does not park a fiber.** Named semaphores have no readiness handle on POSIX, so `wait` blocks the host thread and every fiber on it (`std-lib/concurrency.md` §3). Call it from a dedicated `thread`.

---

## 10. Passing Handles to a Child

Every handle this package creates is **non-inheritable**. A child spawned with `bestie.api.os.spawn` receives only its three stdio streams — unless the parent names more.

Each IPC class offers `token(): HandleToken`. The parent lists tokens in `SpawnOptions.pass` (`os.md` §5.1); the child receives them **at the same numeric value**, learns the values through its arguments or environment, and adopts them:

```bestie
fun adoptReader(tok: HandleToken): own PipeReader ! IpcError
fun adoptWriter(tok: HandleToken): own PipeWriter ! IpcError
fun adoptSocket(tok: HandleToken): own LocalSocket ! IpcError
fun adoptShared(tok: HandleToken, access: Access): own SharedMemory ! IpcError
```

Rules:

* `pass` applies to **that one child only**. Inheritance is not a flag flipped on the handle, which would race with any other thread spawning concurrently and leak the handle into that child too.
* Adopting checks the handle's kind — a token for shared memory adopted as a pipe is `IpcError.InvalidHandle`.
* A token may be adopted once. A second adopt of a token that is still open in this process is `IpcError.InvalidHandle` — two owners of one handle would be a double close.
* The parent should `close()` its copy of an end it passed, or the child will never see end-of-stream on a pipe whose other end the parent still holds open.

### 10.1 Example — a worker reporting on a private pipe

```bestie
import bestie.api.os.spawn
import bestie.api.os.SpawnOptions
import bestie.api.os.Stdio
import bestie.api.io.TextReader
import bestie.api.io.Encoding

fun runWorker(job: str): str ! (IpcError | OsError | ReadError) {
    val own p = try pipe()
    defer p.free()
    val own rd = p.reader else { return IpcError.Closed }
    val own wr = p.writer else { return IpcError.Closed }

    var passed = list<HandleToken>.new()
    passed.add(wr.token())

    val own opts = SpawnOptions.new(
        stdin: Stdio.Null, stdout: Stdio.Inherit, stderr: Stdio.Inherit,
        workingDir: absent, env: absent,
        pass: move passed
    )
    defer opts.freeDeep()

    val args : array<str>[] = {job, wr.token().toStr()}   // the child parses and adopts the token

    val own proc = try spawn("/usr/libexec/worker", args[..], opts.address())
    defer proc.free()
    wr.close()                          // only the child holds the write end now

    val own text = TextReader.new(move rd, Encoding.Utf8)
    defer text.close()
    val report = try text.readAll()     // ends when the child exits or closes its end
    try proc.wait()
    return report
}
```

The report channel is separate from the child's stdout, so a worker that also logs to stdout cannot corrupt the protocol.

---

## 11. Error Model

```bestie
errors IpcError {
    NotFound,           // no object or listener with this name or path
    AlreadyExists,      // exclusive create of a name that exists
    AddressInUse,       // something already exists at the socket path
    Refused,            // a path exists but nothing is listening
    PermissionDenied,   // audience, access mode, or privilege
    InvalidName,        // not a portable IpcName
    NameTooLong,        // socket path exceeds the target's limit
    InvalidHandle,      // wrong kind, or already adopted
    Misaligned,         // offset does not satisfy alignOf(T)
    OutOfRange,         // offset + sizeOf(T) exceeds the mapping
    TimedOut,
    Interrupted,
    ResourceExhausted,  // out of descriptors, memory, or semaphore count
    Closed,             // operation on a closed handle, or a pipe end already moved out
    Unsupported         // the target cannot honor this operation as specified
}
```

Byte-channel failures reuse `ReadError` / `WriteError` from `bestie.api.io`, so a pipe, a local socket, a TCP socket, and a file fail the same way and one handler covers all of them. Error sets compose (`core/exceptions.md` §3.1).

Rules:

* No exceptions, no implicit retries. `Interrupted` is returned; retrying is the caller's policy.
* Peer exit is **not** an error on a read: it is end of stream. It is `WriteError.Closed` on a write, never a signal.

---

## 12. Concurrency Model

`bestie.api.ipc` spawns no threads. Pipes and local sockets honor the fiber parking contract (`std-lib/concurrency.md` §3) exactly as sockets do, so a fiber-per-client daemon over a `LocalListener` is not a thread-per-client daemon. Semaphore waits do not (§9).

Handles are `own`, so moving an accepted `LocalSocket` into a fiber consumes it, and the compiler proves no two workers serve one client — the same proof as for `TcpSocket` (`network.md` §10).

---

## 13. Platform Extensions

Mechanisms without a portable meaning live under:

```text
bestie.api.ipc.unix       // FIFOs, datagram and seqpacket local sockets, socketpair,
                          // descriptor passing (SCM_RIGHTS), peer uid/gid
bestie.api.ipc.linux      // abstract socket namespace, memfd, futex waits on shared
                          // atomics, robust process-shared mutexes, POSIX message queues
bestie.api.ipc.windows    // named pipes, DuplicateHandle, Global\ namespace, ACLs
```

They may add mechanisms and options. They may not change what a `PipeReader`, `LocalSocket`, `SharedMemory`, or `Semaphore` from this package means.

---

## 14. Relationship to Other Layers

| Concern | Package |
| ------- | ------- |
| Stream protocols, buffering, text | `bestie.api.io` |
| Spawning, `HandleToken`, `SpawnOptions.pass`, signals | `bestie.api.os` |
| Socket paths, removing stale files | `bestie.api.fs` |
| `Shutdown`; communication across machines | `bestie.api.network` |
| `@repr(C)` | `bestie.api.foreign` |
| `atomic<T>`, `Ordering` | `bestie.lib.concurrency` |
| `Duration` | `bestie.lib.datetime` |
| Encoding messages | `bestie.lib.format` |

---

## 15. Summary

`bestie.api.ipc` gives a systems program every portable way for local processes to cooperate, and nothing that would mean something different on another OS:

* **Pipes and local sockets are streams.** Buffering, text decoding, and `copy` work across a process boundary with no IPC-specific code, and a dead peer is an ordinary end of stream.
* **No handle reaches a child by accident.** Everything is created non-inheritable, and `SpawnOptions.pass` hands specific handles to one specific child, without the race of a global inherit flag.
* **Shared memory is checked for the boundary it crosses.** Only fixed-width primitives, lock-free atomics, and `@repr(C)` types may be placed in it — anything holding an address, or a layout another build could disagree with, is a compile error.
* **Shared memory hands out `ptr<T>`, not `slice<var T>`,** because exclusivity is a promise the compiler cannot keep about another process. Synchronization is atomics and semaphores, written explicitly.
* **Named objects outlive programs, so creation is exclusive and removal is explicit.** No stale object is silently adopted, and no name is silently unlinked.
* **What a target cannot do, it refuses.** A timed semaphore wait on macOS is `Unsupported`, not a polling loop; owner-death recovery is an extension, not a pretense.

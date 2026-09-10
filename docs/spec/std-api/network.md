# bestie.api.network — Networking API

This document defines the **Bestie Standard Networking API (`bestie.api.network`)**.

`bestie.api.network` provides **raw, explicit, protocol-agnostic networking primitives**.
It is designed for:

* Network services
* Distributed systems
* Infrastructure components
* Protocol implementations (HTTP, gRPC, custom)

This API deliberately avoids:

* High-level protocols
* Hidden concurrency
* Implicit buffering
* Framework abstractions

---

## 1. Scope and Non-Goals

### 1.1 What This API Provides

* TCP and UDP sockets, implementing the stream protocols of `bestie.api.io`
* IP addresses and socket addresses as value types
* Explicit name resolution
* Explicit connect / bind / listen / shutdown
* Explicit socket options, including timeouts
* Blocking I/O primitives with deterministic behavior

---

### 1.2 What This API Does *Not* Provide

* HTTP (see `bestie.api.http`)
* TLS (see `bestie.api.http` §9, or an external package)
* Async runtimes
* Event loops
* Protocol parsers
* Connection pooling frameworks
* Its own buffer type — buffers are core `array<T>` and `slice<T>` (§7)

---

## 2. Design Principles

1. **One abstraction per concept**
2. **Classes for stateful connections, value types for addresses**
3. **Functions for stateless utilities**
4. **No hidden threads**
5. **No implicit buffering**
6. **Ownership governs safety**
7. **Every operation that can fail says so** — no silent `-1` returns

---

## 3. Namespacing

```text
bestie.api.network
```

---

## 4. Class Kinds and Ownership Rationale

| Type | Kind | Why |
| ---- | ---- | --- |
| `IpAddress` | `enum` | A closed set: an address is v4 or v6, never both, never neither. Payload variants carry the octets inline — no heap, no tag word beyond the discriminant. |
| `SocketAddress` | `value class` | An address plus a port is a pure aggregate with no identity. Two socket addresses with the same bytes *are* the same endpoint, so structural equality is what you want, and it copies inline into every call. |
| `TcpSocket` | `class` | A live file descriptor. It has identity (two sockets to the same peer are different connections), owns an OS resource, and its read position is mutable state. |
| `TcpListener` | `class` | Same — owns a bound descriptor. |
| `UdpSocket` | `class` | Same. |
| `SocketOptions` | `value class` | A settings bundle. No identity, copied by value, cheap to pass. |
| `Shutdown` | `enum` | A closed set of three directions. |

Sockets own their descriptors, so every socket is an `own` value and `close()` is the explicit discharge (`core/memory.md` §7). None of them is copyable — a copied descriptor would be a double close.

---

## 5. Addresses

### 5.1 `IpAddress`

```bestie
enum IpAddress {
    V4(uint32),              // host byte order; formatting handles presentation
    V6(array<byte>[16])
}
```

`IpAddress` is a **resolved** address. It never holds a hostname — that distinction is the whole point of §5.3.

### 5.2 `SocketAddress`

```bestie
type Port as uint16 in 1..=65535

value class SocketAddress {
    addr: IpAddress
    port: Port
}
```

`Port` is a range-constrained newtype (`core/lang.md` §6.2), so an out-of-range port is rejected once at construction and never re-checked:

```bestie
val local = SocketAddress.new(
    addr: IpAddress.V4(0x7F000001),      // 127.0.0.1
    port: try (8080 as Port)
)
```

Rules:

* `SocketAddress` is a `value class` — copied inline, structurally compared, no allocation
* It is always a resolved endpoint, never a name awaiting lookup

### 5.3 Name Resolution

DNS is a **network operation**, so it is a function that can fail and can block — never a hidden step inside `connect`:

```bestie
fun resolve(host: str, port: Port): own list<SocketAddress> ! NetError
```

```bestie
val own addrs = try resolve("example.com", try (443 as Port))
defer addrs.freeDeep()
```

Rules:

* Resolution returns **every** address the name maps to, in the order the resolver supplied. Picking one — and falling back to the next — is the caller's decision, because the right policy differs per application.
* An empty result is `NetError.NotFound`, not an empty list. A name that resolves to nothing is a failure, not a success with no answers.
* `connect` never resolves. It takes a `SocketAddress`, so a program that never calls `resolve` performs no DNS traffic and links no resolver.

---

## 6. TCP

### 6.1 `TcpSocket`

A connected TCP stream. It **implements the `bestie.api.io` stream protocols**, so everything written against those works over a socket unchanged:

```bestie
import bestie.api.io.InputStream
import bestie.api.io.OutputStream

class TcpSocket impl InputStream, OutputStream {
    fun read(dst: slice<var byte>): int ! ReadError
    fun write(src: slice<byte>): int ! WriteError
    fun flush(): void ! WriteError

    fun localAddress(): SocketAddress
    fun peerAddress(): SocketAddress

    fun setOptions(opts: SocketOptions): void ! NetError
    fun shutdown(how: Shutdown): void ! NetError
    fun close(): void
}
```

Because it implements the protocols, `io.copy` moves bytes between a socket and a file with no networking-specific code:

```bestie
import bestie.api.io.copy

fun download(sock: ptr<TcpSocket>, out: ptr<File>): int ! IoError {
    return try copy(sock, out)
}
```

Properties:

* Blocking by default, with timeouts set explicitly through `SocketOptions`
* Explicit lifetime, no auto-close
* No buffering — wrap in `BufferedReader` / `BufferedWriter` from `bestie.api.io` when you want it
* A `read` returning `0` means the peer closed its side. That is end of stream, not an error (`io.md` §5).

### 6.2 Connecting

```bestie
fun connect(addr: SocketAddress): own TcpSocket ! NetError
fun connect(addr: SocketAddress, timeout: Duration): own TcpSocket ! NetError
```

Rules:

* Returns an **owned** socket — the caller discharges it with `close()`
* No retries, and no timeout unless one is passed. The two overloads exist so that "no timeout" is a visible choice rather than a forgotten parameter.

### 6.3 Listening

```bestie
class TcpListener {
    fun accept(): own TcpSocket ! NetError
    fun localAddress(): SocketAddress
    fun setOptions(opts: SocketOptions): void ! NetError
    fun close(): void
}

fun listen(addr: SocketAddress, backlog: int): own TcpListener ! NetError
```

`accept()` blocks until a connection arrives and hands the caller an owned socket. Because each accepted socket is `own`, it can be **moved into a fiber or thread** and the compiler proves no two workers hold the same connection (§10).

### 6.4 `Shutdown`

```bestie
enum Shutdown {
    Read,      // no further reads; peer may still receive
    Write,     // send FIN; peer sees end of stream
    Both
}
```

`shutdown(Shutdown.Write)` is how a protocol says "I am done sending" without tearing down the connection — a half-close. `close()` releases the descriptor and cannot be used for that.

---

## 7. Buffers — There Is No `ByteBuffer`

This package defines **no buffer type**. Buffers are core types:

| Need | Type |
| ---- | ---- |
| Owned, fixed-capacity storage | `array<byte>[n]` |
| A read-only view to send | `slice<byte>` |
| A writable view to receive into | `slice<var byte>` |

```bestie
var buf : array<byte>[4096] = array<byte>[4096].fill(0)

val n = try sock.read(buf[..])          // fills the view, returns bytes read
try sock.write(buf[0..n])               // sends exactly what was read
```

A dedicated `ByteBuffer` class would be `slice<T>` with a different name: a base pointer, a length, no allocation, no auto-resize, explicit ownership. Core already provides that, `bestie.api.io` already speaks it, and a second vocabulary would mean every buffer crossing between packages needs a conversion that copies or reinterprets. Where a growable buffer is genuinely needed, that is `list<byte>` from `bestie.lib.collections`.

---

## 8. UDP

```bestie
class UdpSocket {
    fun send(to: SocketAddress, src: slice<byte>): int ! WriteError
    fun receive(dst: slice<var byte>): (int, SocketAddress) ! ReadError

    fun localAddress(): SocketAddress
    fun setOptions(opts: SocketOptions): void ! NetError
    fun close(): void
}

fun bind(addr: SocketAddress): own UdpSocket ! NetError
```

`receive` returns a **tuple** of the byte count and the sender's address (`core/types.md` §4) rather than filling an out-parameter. Both results are produced by the same call, so returning both is the honest signature — and `SocketAddress` is a `value class`, so the tuple costs nothing.

```bestie
val (n, from) = try sock.receive(buf[..])
try sock.send(from, buf[0..n])          // echo it back
```

Rules:

* Message-oriented: one `send` is one datagram, one `receive` is one datagram
* A datagram larger than `dst` is **truncated**, and the returned count reflects what was written. This is the OS behavior and the API does not hide it.
* No delivery guarantees, no ordering guarantees, no retries
* `UdpSocket` deliberately does **not** implement the stream protocols — a datagram socket has no byte stream to model, and pretending otherwise would let `copy` silently reframe message boundaries.

---

## 9. Socket Options

```bestie
value class SocketOptions {
    readTimeout:  Duration ?     // absent means block indefinitely
    writeTimeout: Duration ?
    keepAlive:    bool
    noDelay:      bool           // disable Nagle's algorithm
    reuseAddress: bool
}
```

Every option is explicit and every default is visible at the construction site — there is no ambient configuration and no environment variable that changes socket behavior.

```bestie
import bestie.lib.datetime.Duration
import bestie.lib.datetime.Seconds

try sock.setOptions(SocketOptions.new(
    readTimeout:  Duration.new(30 as Seconds),
    writeTimeout: Duration.new(30 as Seconds),
    keepAlive:    true,
    noDelay:      true,
    reuseAddress: false
))
```

A timeout that expires is `NetError.TimedOut` from the operation that was waiting — never a partial success and never a silent retry.

---

## 10. Concurrency Model

`bestie.api.network` does not spawn threads, does not manage fibers, and provides no async API. Concurrency comes from core `thread` and `bestie.lib.concurrency`, and **ownership is what makes it safe**:

```bestie
import bestie.lib.concurrency.fiber

fun serve(addr: SocketAddress): void ! NetError {
    val own listener = try listen(addr, backlog: 128)
    defer listener.close()

    while (true) {
        val own conn = try listener.accept()

        // The connection is moved into the fiber. The loop cannot touch it
        // afterwards -- the compiler rejects any use of a moved-from binding.
        fiber.new([move conn]() => {
            defer conn.close()
            handle(conn.address()) catch |e| { logFailure(e) }
        })
    }
}
```

Because `accept()` returns an `own TcpSocket`, two workers cannot share one connection by accident: handing it to a fiber consumes it. That is a compile-time guarantee, not a convention.

A blocking `read` on a fiber parks that fiber and releases its host thread, so a fiber-per-connection server is not a thread-per-connection server (`io.md` §8).

---

## 11. Error Model

```bestie
errors NetError {
    NotFound,          // resolution produced no addresses
    Refused,           // connection actively refused
    Unreachable,       // no route to host
    TimedOut,
    AddressInUse,
    PermissionDenied,
    Closed,            // operation on a closed socket
    ResolutionFailed
}
```

Read and write failures reuse `ReadError` / `WriteError` from `bestie.api.io`, so a socket and a file fail the same way and one handler covers both. `errors IoError = ReadError | WriteError` composes per `core/exceptions.md` §3.1, so `try` propagates across the boundary with no conversion.

Rules:

* No exceptions, no implicit retries, no `-1` sentinels
* Every fallible operation declares its error set
* Peer disconnection is **not** an error: a `read` of `0` is end of stream (§6.1)

---

## 12. Platform Extensions

Platform-specific features live under:

```text
bestie.api.network.linux
bestie.api.network.bsd
```

They must not alter the semantics defined here — an extension may add an option or expose a raw descriptor, never change what `read` or `connect` means.

---

## 13. Summary

`bestie.api.network` is raw, explicit, deterministic, and minimal:

* **Addresses are value types.** `SocketAddress` is a `value class` over an `IpAddress` enum and a range-constrained `Port` — copied inline, structurally compared, always resolved.
* **DNS is a visible operation.** `resolve()` can fail and can block; `connect` never resolves, so a program that does no lookups links no resolver.
* **Sockets are streams.** `TcpSocket` implements the `bestie.api.io` protocols, so buffering, text decoding, and `copy` all work over the network with no networking-specific code.
* **There is no buffer type here.** `array<byte>`, `slice<byte>`, and `slice<var byte>` are core, and one vocabulary crosses every package boundary without conversion.
* **Ownership is the concurrency proof.** `accept()` returns `own`, so moving a connection into a worker consumes it and no two workers can hold the same one.
* **Everything that can fail says so.** No `-1`, no errno, no silent truncation of a failure into a count.

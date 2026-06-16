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

* TCP and UDP sockets
* Address handling
* Explicit connect / bind / listen
* Blocking I/O primitives
* Deterministic behavior

---

### 1.2 What This API Does *Not* Provide

* HTTP (see `bestie.api.http`)
* TLS (separate API or external)
* Async runtimes
* Event loops
* Protocol parsers
* Connection pooling frameworks

---

## 2. Design Principles

1. **One abstraction per concept**
2. **Classes for stateful connections**
3. **Functions for stateless utilities**
4. **No hidden threads**
5. **No implicit buffering**
6. **Ownership governs safety**

---

## 3. Namespacing

```text
bestie.api.network
```

---

## 4. Network Addresses

### 4.1 `SocketAddress`

```bestie
class SocketAddress {
    host: str
    port: int
}
```

Rules:

* No DNS resolution unless explicitly requested
* Immutable value type

---

## 5. TCP

### 5.1 `TcpSocket`

Represents a connected TCP stream.

```bestie
class TcpSocket {
    fun read(buffer: ref ByteBuffer): int
    fun write(buffer: ref ByteBuffer): int
    fun close(): void
}
```

Properties:

* Blocking by default
* Explicit lifetime
* No auto-close
* No buffering guarantees

---

### 5.2 Connecting

```bestie
fun connect(addr: SocketAddress): TcpSocket
```

Rules:

* Errors are explicit
* No retries
* No implicit timeouts

---

### 5.3 Listening

```bestie
class TcpListener {
    fun accept(): TcpSocket
    fun close(): void
}
```

```bestie
fun listen(addr: SocketAddress, backlog: int): TcpListener
```

---

## 6. UDP

### 6.1 `UdpSocket`

```bestie
class UdpSocket {
    fun send(to: SocketAddress, buffer: ref ByteBuffer): int
    fun receive(from: ref SocketAddress, buffer: ref ByteBuffer): int
    fun close(): void
}
```

Rules:

* Message-oriented
* No delivery guarantees
* Explicit addresses

---

## 7. Byte Buffers

Networking APIs rely on explicit buffers:

```bestie
class ByteBuffer {
    data: ptr<byte>
    capacity: int
    length: int
}
```

Rules:

* No hidden allocation
* No auto-resize
* Ownership is explicit

---

## 8. Concurrency Model

`bestie.api.network`:

* Does not spawn threads
* Does not manage fibers
* Does not provide async APIs

Concurrency is achieved using:

* Core language concurrency primitives
* Ownership transfer
* Explicit coordination

---

## 9. Error Model

All operations return:

* Explicit error types
* No exceptions
* No implicit retries

---

## 10. Platform Extensions

Platform-specific features must live under:

```text
bestie.api.network.linux
bestie.api.network.bsd
```

They must not alter core semantics.

---

## 11. Summary

`bestie.api.network` is:

* Raw
* Explicit
* Deterministic
* Minimal

It provides the **ground truth of networking**, upon which higher protocols can be built cleanly.

This document is **finalized**.

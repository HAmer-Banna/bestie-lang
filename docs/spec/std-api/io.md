# Standard API — I/O (`bestie.api.io`)

This document defines the **I/O API layer** for Bestie.

`bestie.api.io` provides **structured, explicit, and composable I/O abstractions** built on top of the Bestie core language.

It intentionally **does not replace** the minimal console I/O that exists in the core language.

---

## 1. Scope and Intent

The purpose of `bestie.api.io` is to provide:

* Stream-based I/O
* Buffered I/O
* Binary and text I/O
* Explicit resource lifetimes
* Composable I/O pipelines

It is designed for:

* Backend services
* Networking layers
* File systems
* CLI tools
* Streaming and data processing

---

## 2. What Is Already in Core (Explicitly)

The **Bestie core language** provides **minimal console I/O**, equivalent to C’s `scanf` / `printf`.

These are:

* Always available
* Zero-configuration
* Synchronous
* Blocking
* Side-effectful by definition

Examples (core, not API):

```bestie
print("Enter name:")
val name = input()
print("Hello ${name}")
```

Rules:

* No streams
* No buffering control
* No async variants
* No extensibility

This is **intentional** and sufficient for:

* Scripts
* REPL
* Experiments
* Serverless handlers
* Debugging

Anything beyond this belongs to `bestie.api.io`.

---

## 3. Design Principles of `bestie.api.io`

`bestie.api.io` follows these rules:

1. **Explicit resource ownership**
2. **No hidden buffering**
3. **No global state**
4. **Blocking by default**
5. **Async via concurrency, not magic**
6. **No exceptions**
7. **Composable primitives only**

I/O errors are expressed using **core error unions**, not exceptions.

---

## 4. Core Abstractions

### 4.1 Streams

All I/O is modeled using **streams**.

```bestie
InputStream
OutputStream
```

Properties:

* Explicit open / close
* Ownership-aware
* Not implicitly thread-safe
* Deterministic lifetime

---

### 4.2 Byte vs Text Streams

Two primary stream families exist:

* **Binary streams**
* **Text streams**

```bestie
ByteInputStream
ByteOutputStream

TextInputStream
TextOutputStream
```

Rules:

* No implicit encoding conversion
* Text streams require explicit encoding
* UTF-8 is the default, not the only option

---

## 5. Reading and Writing

### 5.1 Reading

```bestie
fun read(): byte[] | IOError
fun readExact(n: int): byte[] | IOError
```

Rules:

* Partial reads are explicit
* EOF is not an error
* Blocking semantics are explicit

---

### 5.2 Writing

```bestie
fun write(data: byte[]): void | IOError
fun flush(): void | IOError
```

No implicit flushing occurs.

---

## 6. Buffering

Buffering is **explicit** and opt-in.

```bestie
BufferedInputStream
BufferedOutputStream
```

Rules:

* Buffer size must be specified
* Buffering does not change semantics
* Buffering never hides errors

---

## 7. Resource Management

I/O resources:

* Are not garbage-collected
* Must be closed explicitly
* Follow ownership rules

Example:

```bestie
own stream = FileInputStream.open(path)
defer stream.close()

process(stream)
```

(Exact `defer` semantics may be finalized later, but resource lifetime is always explicit.)

---

## 8. Integration with Concurrency

`bestie.api.io` does **not** introduce async keywords.

Concurrency is achieved via:

* core `thread`
* `fiber` from `bestie.lib.concurrency`

Example:

```bestie
import bestie.lib.concurrency.fiber

fiber.of(() => {
    own data = stream.read()
    process(data)
})
```

This keeps:

* I/O explicit
* Scheduling visible
* Semantics predictable

---

## 9. Error Handling

All I/O functions return **typed errors**:

```bestie
ReadError
WriteError
IOError
```

Rules:

* No exceptions
* No hidden retries
* Errors must be handled or propagated

---

## 10. What `bestie.api.io` Explicitly Excludes

`bestie.api.io` does **not** include:

* File system operations (→ `bestie.api.fs`)
* Networking (→ `bestie.api.network`)
* HTTP (→ `bestie.api.http`)
* Compression formats
* Serialization formats (JSON, CSV, etc.)
* Async/await
* Event loops

Each concern has its own API layer.

---

## 11. Relationship to Other APIs

* `bestie.api.fs` builds on `bestie.api.io`
* `bestie.api.network` builds on `bestie.api.io`
* `bestie.api.http` builds on `bestie.api.network`
* Higher-level frameworks build on all of the above

This ensures **clean layering and testability**.

---

## 12. Stability and Evolution

Rules:

* I/O APIs evolve conservatively
* Breaking changes require major version bumps
* No feature is added unless it cannot be expressed cleanly using streams

---

## 13. Summary

* Core language provides **minimal console I/O**
* `bestie.api.io` provides **structured stream-based I/O**
* No overlap, no duplication
* Explicit, safe, composable, and predictable

This clean separation keeps the core sealed and the ecosystem extensible.

---

**Next step**: `bestie.api.os.md`
(Process management, environment variables, signals, clocks)

# Standard API — I/O (`bestie.api.io`)

This document defines the **I/O API layer** for Bestie.

`bestie.api.io` is how Bestie **talks to the outside world** for I/O: console, files, and streams. String interpolation (`"Hello ${name}"`) is core syntax. Codecs (JSON, XML, …) are `bestie.lib.format`.

---

## 1. Scope and Intent

The purpose of `bestie.api.io` is to provide:

* Hosted console I/O (`print`, `println`, `input`, `printf`)
* The `InputStream` / `OutputStream` protocols every other I/O package implements
* `stdin` / `stdout` / `stderr` as ordinary stream values
* Explicit buffering and explicit text encoding
* Explicit resource lifetimes
* Composable I/O pipelines

It is designed for:

* Backend services
* Networking layers
* File systems
* CLI tools
* Streaming and data processing

---

## 2. Hosted Console

```bestie
import bestie.api.io.println
import bestie.api.io.print
import bestie.api.io.input
import bestie.api.io.printf

fun main() {
    println("Hello, Bestie")
    print("Enter name: ")
    val name = input()
    println("Hello ${name}")
    printf("age %d\n", 30)
}
```

Synchronous and blocking. No streams, no buffering control, no async. Freestanding / bare-metal programs do not use this surface — they use MMIO or other `bestie.api` packages.

`printf` uses C format specifiers (`%s`, `%d`, `%f`, `%x`, …), type-checked when the format string is a literal.

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

## 4. Class Kinds and Ownership Rationale

| Type | Kind | Why |
| ---- | ---- | --- |
| `InputStream` | `protocol` | Many unrelated things are readable — a `File`, a `TcpSocket`, an in-memory buffer. A protocol lets one `copy` serve all of them with static dispatch and no vtable. |
| `OutputStream` | `protocol` | Same, for writing. |
| `TextReader` | `class` | Wraps a byte stream plus encoder state. Stateful (a multi-byte scalar can straddle a read boundary) and has identity. |
| `TextWriter` | `class` | Same, for encoding out. |
| `BufferedReader` | `class` | Owns a heap buffer and a cursor. Mutable state, identity. |
| `BufferedWriter` | `class` | Same, plus unflushed data — losing track of *which* writer holds it would be a correctness bug, so identity is essential. |
| `Encoding` | `enum` | A closed set of text encodings. No state. |

Every wrapper **owns** the stream it wraps, so wrapper classes declare `own` fields and `close()` releases the whole chain. Ownership flows one way: a wrapper takes ownership of what it wraps, which is why `move` is required to build one.

---

## 5. Stream Protocols

All I/O is modelled on **two protocols**. Everything else in this document is a class that implements or wraps them.

```bestie
protocol InputStream {
    /// Reads into `dst`, returning the number of bytes read.
    /// A return of 0 means end of stream — it is not an error.
    fun read(dst: slice<var byte>): int ! ReadError

    fun close(): void
}

protocol OutputStream {
    /// Writes from `src`, returning the number of bytes accepted.
    /// A short write is normal and is not an error.
    fun write(src: slice<byte>): int ! WriteError

    fun flush(): void ! WriteError
    fun close(): void
}
```

**Why two protocols and not four.** Splitting byte and text streams into separate protocol pairs is the obvious alternative, and it is wrong: text is not a different kind of stream, it is a byte stream plus an encoding. A separate pair would duplicate every method for one field's worth of difference. Encoding is a **wrapper** instead (§6), which also makes "no implicit encoding conversion" structural rather than a rule to remember.

**Why protocols and not classes.** A `File` and a `TcpSocket` genuinely *are* readable. If streams were classes, their `read` methods would be unrelated functions that merely share a name, and no generic operation over "something readable" could be written at all:

```bestie
fun <S impl InputStream, D impl OutputStream> copy(src: ptr<S>, dst: ptr<D>): int ! IoError {
    var buf : array<byte>[8192] = array<byte>[8192].fill(0)
    var total = 0

    while (true) {
        val n = try src.val.read(buf[..])
        if (n == 0) { break }

        var written = 0
        while (written < n) {
            written += try dst.val.write(buf[written..n])
        }
        total += n
    }
    return total
}
```

Protocol dispatch is static by default (`core/oop.md` §2.2), so this monomorphizes per pair — no vtable, no boxing — and both type parameters are inferred from the arguments (`core/lang.md` §7.1).

### 5.1 `copy`

`copy` is the one utility the protocols exist to make possible, and it ships with the package:

```bestie
fun <S impl InputStream, D impl OutputStream> copy(src: ptr<S>, dst: ptr<D>): int ! IoError
fun <S impl InputStream, D impl OutputStream> copy(src: ptr<S>, dst: ptr<D>, bufferSize: int): int ! IoError
```

It reads until end of stream, writes everything it reads, and returns the total byte count. It allocates a stack buffer, never a heap one, and it does not flush or close either side — those remain the caller's explicit decisions.

Because the parameters are protocol-constrained, one function serves every pair: file to socket, socket to file, buffer to `stdout`. Each instantiation monomorphizes into a direct call.

**Rules:**

* `read` returning `0` is end of stream. EOF is **not** an error, so it needs no error variant and no separate `eof()` query.
* Short reads and short writes are normal. Callers loop, as `copy` does above; nothing retries on your behalf.
* `read` takes `slice<var byte>` — the caller owns the buffer and the stream writes through the view, so no allocation happens per call (`core/types.md` §6).
* Streams are **not** implicitly thread-safe. Sharing one across threads needs ownership transfer or a `Lock`.
* `close()` is explicit and idempotent. Nothing closes at scope exit — pair it with `defer`.

---

## 6. Standard Streams and Text

### 6.1 `stdin`, `stdout`, `stderr`

The three standard streams are values, so they compose with everything above:

```bestie
import bestie.api.io.stdout
import bestie.api.io.stderr

val stdin  : InputStream
val stdout : OutputStream
val stderr : OutputStream
```

They are process-lifetime handles that the program does not own: **do not `close()` them**, and their ownership is never transferred. Closing a standard stream is a compile-time error.

The console functions of §2 are built on these, and having them as values is what makes redirection possible:

```bestie
fun report(out: ptr<OutputStream>, msg: str): void ! WriteError {
    try out.val.write(msg.bytes())
}

report(stdout.address(), "done\n")     // to the console
report(logFile.address(), "done\n")    // same function, to a file
```

### 6.2 `Encoding`

```bestie
enum Encoding {
    Utf8,
    Utf16Le,
    Utf16Be,
    Ascii,
    Latin1
}
```

### 6.3 `TextReader` / `TextWriter`

Text is a byte stream plus an encoding, and the wrapping is always explicit:

```bestie
class TextReader {
    val own source: InputStream
    val encoding: Encoding

    fun readLine(): str ? ! ReadError    // absent at end of stream
    fun readAll(): str ! ReadError
    fun close(): void                    // closes the wrapped stream
}

class TextWriter {
    val own sink: OutputStream
    val encoding: Encoding

    fun write(text: str): void ! WriteError
    fun writeLine(text: str): void ! WriteError
    fun flush(): void ! WriteError
    fun close(): void
}
```

`readLine()` returns `str ? ! ReadError` — a line, a clean end of stream, or a read failure. All three are distinct outcomes, which is exactly the shape `core/types.md` §8.4 permits, so a line-oriented reader iterates with `try for` (`core/lang.md` §13).

**Rules:**

* Constructing a `TextReader` **moves** the byte stream into it. The source is no longer separately usable, which is what prevents interleaved raw and decoded reads from desynchronising the decoder.
* Invalid bytes for the declared encoding are a `ReadError`, never a silent replacement character.
* UTF-8 is the conventional default but never the implicit one — the encoding is always written at the construction site.

---

## 7. Buffering and Resource Management

### 7.1 Buffering is explicit

```bestie
class BufferedReader impl InputStream {
    val own source: InputStream

    fun read(dst: slice<var byte>): int ! ReadError
    fun close(): void
}

class BufferedWriter impl OutputStream {
    val own sink: OutputStream

    fun write(src: slice<byte>): int ! WriteError
    fun flush(): void ! WriteError
    fun close(): void
}
```

Because both implement the stream protocols, a buffer is transparent to everything downstream — `copy` neither knows nor cares that one is present.

Rules:

* Buffer capacity is a required constructor argument. There is no default size to be surprised by.
* Buffering changes performance, never semantics.
* Buffering never hides an error: a failure during an internal flush surfaces from the `write` or `flush` that triggered it.
* **`BufferedWriter.close()` flushes first.** If that flush fails, `close()` still releases the underlying resource and then reports the error — a failed flush must not leak a file descriptor.

### 7.2 Resource management

I/O resources are not garbage collected, are never closed implicitly, and follow ordinary ownership rules (`core/memory.md` §7).

```bestie
fun readConfig(path: Path): str ! IoError {
    val own file = try fs.open(path, OpenMode.Read)
    val own reader = TextReader.new(move file, Encoding.Utf8)
    defer reader.close()

    return try reader.readAll()
}
```

Wrapping transfers ownership, so a chain has exactly one owner at each level and `close()` on the outermost wrapper releases all of it. That is also why `file` cannot be used after the `move` — the compiler rejects it as a use-after-move, which is what makes the single `defer` sufficient.

---

## 8. Integration with Concurrency

`bestie.api.io` does **not** introduce async keywords.

Concurrency is achieved via:

* core `thread`
* `fiber` from `bestie.lib.concurrency`

Example:

```bestie
import bestie.lib.concurrency.fiber

// The fiber takes ownership of the stream — nothing is shared across the boundary.
fiber.new([move stream]() => {
    var buf : array<byte>[4096] = array<byte>[4096].fill(0)
    defer stream.close()

    while (true) {
        val n = stream.read(buf[..]) catch |e| { return }
        if (n == 0) { break }
        process(buf[..n])
    }
})
```

A blocking `read` parks the fiber and releases its host thread; it does not block the scheduler. That is the whole reason `bestie.api.io` needs no async colouring — the blocking call *is* the async call, and which one it is depends on what is running it.

This keeps:

* I/O explicit
* Scheduling visible
* Semantics predictable

---

## 9. Error Handling

All I/O functions return **typed errors**:

```bestie
errors ReadError  { Interrupted, Closed, InvalidEncoding, DeviceFailure }
errors WriteError { Interrupted, Closed, NoSpace, DeviceFailure }
errors IoError = ReadError | WriteError
```

Error sets compose per `core/exceptions.md` §3.1, so a function declaring `! IoError` accepts `try` on anything returning `! ReadError` or `! WriteError` with no conversion.

**There is no `Eof` variant.** End of stream is a normal outcome, reported as a `0` return from `read` (§5) and as absence from `TextReader.readLine()`. Making it an error would force every caller to pattern-match a failure on the most ordinary path in the API — the same mistake as an `End` iterator variant (`core/types.md` §8.4).

Rules:

* No exceptions
* No hidden retries — a short read is returned to you, not looped over internally
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

`bestie.api.io` is the smallest package that everything else talking to the outside world builds on:

* **Two protocols** — `InputStream` and `OutputStream`. `bestie.api.fs` and `bestie.api.network` implement them; nothing re-invents them.
* **`stdin` / `stdout` / `stderr` are values**, not just functions, so the console is a stream like any other and output can be redirected without changing a signature.
* **Text is a wrapper, not a second stream family.** An encoding is always written at the construction site, which makes "no implicit encoding conversion" structural.
* **Buffering is a wrapper too**, transparent to everything downstream because it implements the same protocols.
* **EOF is not an error.** A `0` read and an absent line are ordinary outcomes.
* **Ownership flows one way** — a wrapper owns what it wraps, so one `close()` releases the chain and the compiler rejects any use of the moved-from handle.

Interpolation stays core syntax; structured codecs stay in `bestie.lib.format`, which wins the `format` name over any api package.

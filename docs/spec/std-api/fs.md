# Standard API — File System (`bestie.api.fs`)

This document defines the **File System API layer** for Bestie.

`bestie.api.fs` provides **explicit, structured, and portable access** to files, directories,
and file metadata. It builds directly on [`bestie.api.io`](io.md) for reading and writing,
and on [`bestie.api.os`](os.md) for process- and platform-level concerns.

`bestie.api.fs` is **not** a virtual file system, **not** a watcher framework, and **not** a
path-manipulation DSL. It exposes the file system as the OS presents it, without hiding cost.

---

## 1. Scope and Intent

### 1.1 What `bestie.api.fs` Provides

* Opening files as `bestie.api.io` streams
* Path representation and inspection
* File and directory metadata (`stat`)
* Directory creation, listing, and removal
* File operations: rename, copy, remove, link
* Explicit permission and timestamp queries

### 1.2 What `bestie.api.fs` Does *Not* Provide

* Stream primitives (→ `bestie.api.io`)
* Process / environment access (→ `bestie.api.os`)
* Networking or remote file systems (→ `bestie.api.network`)
* Serialization formats — JSON, CSV, etc.
* File watching / inotify-style events
* Implicit recursive globbing as a query language

Each concern has its own API layer. `bestie.api.fs` is the bridge between paths on disk and
the streams defined in `bestie.api.io`.

---

## 2. Design Principles

1. **Explicit resource ownership** — open handles must be closed; lifetimes are visible.
2. **No hidden buffering** — buffering comes from `bestie.api.io`, opt-in.
3. **No global state** — no current-working-directory mutation hidden in calls.
4. **Blocking by default** — concurrency comes from threads, not async magic.
5. **No exceptions** — every fallible operation returns a typed error.
6. **Portable surface, platform-specific backends** — one API, honest about platform limits.
7. **Paths are values, not strings** — `Path` is a first-class value type.

---

## 3. Namespacing

All file system APIs live under:

```text
bestie.api.fs
```

No re-exports.

```bestie
import bestie.api.fs
```

---

## 4. Class Kinds and Ownership Rationale

| Type | Kind | Reasoning |
| ---- | ---- | --------- |
| `Path` | `value class` | An immutable path value; inlined; no identity; freely copyable |
| `Metadata` | `data class` | A snapshot of file attributes; structural equality is meaningful; immutable |
| `Permissions` | `value class` | A small bitset of mode flags; inlined; no identity |
| `FileType` | `enum` | Closed set of variants (file, dir, symlink, other) |
| `OpenMode` · `SeekFrom` | `enum` | Closed sets; no flag bitmasks |
| `File` | `class` | Holds an OS file descriptor — mutable, identity-bearing, must be closed. Implements the `bestie.api.io` stream protocols (§6) |
| `DirIterator` | `class` | Holds an open directory handle and cursor state; stateful; must be closed |

**Why `File` and `DirIterator` are `class`:**
Both wrap a live OS handle (a file descriptor / directory stream). They carry mutable state
(position, open/closed status) and an identity that must not be duplicated by assignment — a
copied descriptor would alias the same kernel object and double-close it. They therefore
require identity, ownership (`own`), and explicit `close()`.

**Why `Path`, `Metadata`, `Permissions` are value/data types:**
They are immutable snapshots — pure data with no live OS resource. `Path` and `Permissions`
are small and inlined; `Metadata` has structural equality (two stats with the same fields are
equal). None owns a heap resource.

**Field ownership:** `Path` owns its internal string by value. `File` and `DirIterator` own
their OS handle and are the only types here that require `own` at use sites. No field carries a
`ref` qualifier.

---

## 5. Paths

### 5.1 `Path`

```bestie
value class Path {
    val raw: str   // platform-native textual form
}
```

Paths are **values**, compared and combined explicitly — never mutated in place.

Construction is `Path.new`; the query functions are free functions and pure:

```bestie
Path.new(text: str): Path                  // an ordinary init, core/oop.md §13.2

fun join(base: Path, child: str): Path
fun parent(p: Path): Path ?               // absent if p has no parent
fun fileName(p: Path): str ?               // absent for the root path
fun extension(p: Path): str ?
fun isAbsolute(p: Path): bool
fun normalize(p: Path): Path               // lexical only — no disk access
```

Rules:

* Path operations in §5 are **lexical** — they never touch the disk
* No implicit separator guessing; `join` uses the platform separator
* `normalize` resolves `.` / `..` textually, not via the file system

---

## 6. Opening Files

A `File` **is** a stream. It implements the `bestie.api.io` protocols, so everything written against those works on a file with no file-specific code:

```bestie
import bestie.api.io.InputStream
import bestie.api.io.OutputStream

class File impl InputStream, OutputStream {
    fun read(dst: slice<var byte>): int ! ReadError
    fun write(src: slice<byte>): int ! WriteError
    fun flush(): void ! WriteError

    fun metadata(): Metadata ! FsError
    fun sync(): void ! FsError            // flush OS buffers to durable storage
    fun truncate(size: int64): void ! FsError
    fun seek(offset: int64, from: SeekFrom): int64 ! FsError
    fun path(): Path

    fun close(): void
}

enum OpenMode {
    Read,       // read only; fails if missing
    Truncate,   // write; create or empty
    Append,     // write; create or append
    CreateNew,  // write; fail if it already exists
    ReadWrite   // read and write; create if missing
}

enum SeekFrom {
    Start, Current, End
}

fun open(p: Path, mode: OpenMode): own File ! FsError
```

There is **one** way to open a file. An earlier shape offered `openRead` / `openWrite` returning streams *and* a separate `File` handle for metadata and syncing, which meant choosing up front whether you would ever need to `stat` what you had opened. Since `File` implements the stream protocols, the split buys nothing.

### 6.1 Example

```bestie
import bestie.api.io.TextReader
import bestie.api.io.Encoding
import bestie.api.io.copy

fun readConfig(p: Path): str ! (FsError | ReadError) {
    val own f = try open(p, OpenMode.Read)
    val own text = TextReader.new(move f, Encoding.Utf8)
    defer text.close()                    // closes the wrapped file too

    return try text.readAll()
}

fun backup(from: Path, to: Path): int ! (FsError | IoError) {
    val own src = try open(from, OpenMode.Read)
    defer src.close()

    val own dst = try open(to, OpenMode.CreateNew)
    defer dst.close()

    return try copy(src.address(), dst.address())
}
```

Rules:

* A `File` is `own` and must be closed explicitly — nothing closes at scope exit
* Wrapping a `File` in a `TextReader` or `BufferedReader` **moves** it; closing the wrapper closes the file (`bestie.api.io` §7.2)
* No implicit buffering. A raw `File` write is a syscall; wrap in `BufferedWriter` when that matters.
* Reading from a file opened `OpenMode.Truncate`, or writing to one opened `OpenMode.Read`, is `FsError.WrongMode` — the mode is enforced, not advisory

---

## 7. Metadata

### 7.1 `Metadata`

```bestie
data class Metadata {
    val kind: FileType
    val size: int64           // bytes
    val permissions: Permissions
    val modified: Instant       // bestie.lib.datetime value type
    val created:  Instant ?     // absent where the platform does not record it
}
```

```bestie
enum FileType {
    Regular,
    Directory,
    Symlink,
    Other
}
```

Stateless queries (free functions):

```bestie
fun stat(p: Path): Metadata ! FsError       // follows symlinks
fun lstat(p: Path): Metadata ! FsError       // does not follow symlinks
fun exists(p: Path): bool                    // never errors; absence is not an error
```

Rules:

* `Metadata` is an immutable **snapshot** taken at the time of the call
* `exists` returns a plain `bool`; only genuine I/O failures are errors
* Timestamps reuse `Instant` from [`bestie.lib.datetime`](../std-lib/datetime.md) — no bespoke time type
* `created` is `Instant ?`: several filesystems do not record a creation time, and reporting the modification time instead would be a lie

### 7.2 `Permissions`

```bestie
value class Permissions {
    val bits: uint32   // platform mode bits
}
```

* A copyable value snapshot, not a live handle
* Interpretation is platform-defined; helpers expose the portable subset

```bestie
fun isReadOnly(perm: Permissions): bool
fun setReadOnly(p: Path, readOnly: bool): void ! FsError
```

---

## 8. Directories

### 8.1 Creation and Removal

```bestie
fun createDir(p: Path): void ! FsError            // fails if parent is missing
fun createDirAll(p: Path): void ! FsError          // creates intermediate dirs
fun remove(p: Path): void ! FsError                // file or empty dir
fun removeAll(p: Path): void ! FsError             // recursive — explicit, never implicit
```

Rules:

* Recursive removal is a **separate, explicitly named** function (`removeAll`) — never a flag
* `createDir` does not silently create parents; use `createDirAll` for that

### 8.2 Listing — `DirIterator`

Directory listing is an **iterator over a live handle**, not an eagerly materialized list, so
large directories do not force an allocation.

```bestie
class DirIterator {
    fun next(): Path ? ! FsError
    fun close(): void
}

fun readDir(p: Path): own DirIterator ! FsError
```

```bestie
fun scanLogs(): void ! FsError {
    val own dir = try readDir(Path.new("/var/log"))
    defer dir.close()

    try for (entry in dir) {
        process(entry)
    }
}
```

`try for` is required because `next()` is fallible (`core/lang.md` §13): `try` propagates an `FsError` out of the loop and out of `scanLogs`, while a clean end of directory ends the loop normally. Written by hand the same loop is:

```bestie
val it = dir.iterator()
while (true) {
    val entry = try it.next() else { break }
    process(entry)
}
```

Rules:

* `DirIterator` owns an OS handle and must be closed
* `next()` returns absent at end of directory and `! FsError` on a read failure — the two outcomes are genuinely different and both must be expressible. `T ? ! E` is legal (`core/types.md` §8.4); iterate with `try for` (`core/lang.md` §13).
* Iteration order is **not** guaranteed across platforms
* Entries are returned as `Path`; call `stat` if metadata is needed

---

## 9. File Operations

```bestie
fun rename(from: Path, to: Path): void ! FsError       // atomic when same volume
fun copyFile(from: Path, to: Path): void ! FsError     // contents + permissions
fun symlink(target: Path, link: Path): void ! FsError
fun readLink(link: Path): Path ! FsError
```

Rules:

* `rename` is atomic on the same volume; cross-volume moves are an error, not a silent copy
* `copyFile` never follows into directories implicitly — it copies a single regular file. It is named `copyFile` rather than `copy` so it does not collide with `bestie.api.io.copy`, which streams between any two handles
* Symlink support may be conditionally unavailable on some platforms (reported via `FsError`)

---

## 10. Error Model

All fallible operations return a **typed error** via the core error union (`!`), consistent
with `bestie.api.io` and `bestie.api.os`.

```bestie
errors FsError {
    NotFound,
    PermissionDenied,
    AlreadyExists,
    NotADirectory,
    IsADirectory,
    NotEmpty,
    CrossDevice,        // e.g. rename across volumes
    WrongMode,          // read on a write-only handle, or the reverse
    Unsupported,        // operation not available on this platform
    Io                  // underlying bestie.api.io failure
}
```

Rules:

* No exceptions
* No hidden retries or silent fallbacks
* Absence (`exists` == false) is **not** an error; only failed operations are
* `FsError.Io` wraps lower-level `bestie.api.io` errors without losing the typed boundary

---

## 11. Integration with Concurrency

`bestie.api.fs` introduces **no** async keywords. Blocking calls run on whatever thread invokes
them; concurrency is achieved with core `thread` or `fiber` from `bestie.lib.concurrency` (matching `bestie.api.io`):

```bestie
import bestie.lib.concurrency.fiber

fun scanInBackground(p: Path): void ! FsError {
    val own f = try open(p, OpenMode.Read)

    // The handle is moved into the fiber, which owns and closes it.
    fiber.new([move f]() => {
        defer f.close()

        var buf : array<byte>[4096] = array<byte>[4096].fill(0)
        while (true) {
            val n = f.read(buf[..]) catch |e| { return }
            if (n == 0) { break }
            process(buf[..n])
        }
    })
}
```

* File handles are **not** implicitly thread-safe
* A handle should be owned and used by one thread at a time

---

## 12. Relationship to Other APIs

* `bestie.api.fs` **builds on** `bestie.api.io` — a `File` *is* an `InputStream` / `OutputStream`, so this package defines no read or write surface of its own
* `bestie.api.fs` **uses** `bestie.api.os` for platform metadata and `bestie.lib.datetime` for timestamps
* `bestie.api.network` and `bestie.api.http` are independent peers, not built on `fs`

This preserves the clean layering described in `bestie.api.io`.

---

## 13. What `bestie.api.fs` Explicitly Excludes

* File watching / change notifications
* Glob / pattern-matching query language
* Memory-mapped files (→ `bestie.api.memory`)
* Temp-file lifecycle frameworks
* Implicit current-working-directory state
* Serialization or encoding of file contents

---

## 14. Stability and Evolution

* APIs are additive within a major version
* Platform-specific behavior is reported through `FsError`, never hidden
* No feature is added unless it cannot be expressed cleanly via streams + paths

---

## 15. Summary

`bestie.api.fs` is:

* Explicit — handles are owned and closed; paths are values
* Portable — one surface, honest about platform limits
* Composable — a `File` *is* a stream, so `copy`, `BufferedReader`, and `TextReader` all work on it unchanged
* Predictable — typed errors, no exceptions, no hidden state

`File` and `DirIterator` are the only classes — the types that own live OS handles. `Path`,
`Permissions`, and `Metadata` are value/data snapshots. The file system is exposed as it is,
**without leaking OS chaos into the language**.

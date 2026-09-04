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
| `File` | `class` | Holds an OS file descriptor — mutable, identity-bearing, must be closed |
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

Factory and query functions (free functions, pure):

```bestie
fun Path.of(text: str): Path
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

Files are opened into `bestie.api.io` streams. `bestie.api.fs` provides the openers; `bestie.api.io`
provides the read/write surface.

```bestie
fun openRead(p: Path): own ByteInputStream ! FsError
fun openWrite(p: Path, mode: OpenMode): own ByteOutputStream ! FsError
```

```bestie
enum OpenMode {
    Truncate,   // create or empty
    Append,     // create or append
    CreateNew   // fail if it already exists
}
```

Example (ownership and explicit close, consistent with `bestie.api.io`):

```bestie
own stream = openRead(Path.of("/etc/config")) catch |err| { ... }
defer stream.close()

own data = stream.read()
process(data)
```

Rules:

* Openers return `bestie.api.io` streams — `bestie.api.fs` adds no new read/write methods
* Handles are owned (`own`) and must be closed explicitly
* No implicit buffering — wrap in `BufferedInputStream` from `bestie.api.io` if desired

---

## 7. The `File` Handle

For operations that go beyond plain streaming (querying live metadata, syncing, truncating),
`bestie.api.fs` exposes an explicit handle.

```bestie
class File {
    fun metadata(): Metadata ! FsError
    fun sync(): void ! FsError          // flush OS buffers to durable storage
    fun truncate(size: int64): void ! FsError
    fun close(): void
}
```

Construction uses static factory methods (`@noNew`):

```bestie
own f = File.open(Path.of("data.bin"), OpenMode.Append) catch |err| { ... }
defer f.close()
```

`File` is a `class` because it owns a mutable OS descriptor with identity. It is never copied;
ownership is transferred explicitly.

---

## 8. Metadata

### 8.1 `Metadata`

```bestie
data class Metadata {
    val kind: FileType
    val size: int64           // bytes
    val permissions: Permissions
    val modified: Instant     // bestie.lib.datetime value type
    val created: Instant
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

### 8.2 `Permissions`

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

## 9. Directories

### 9.1 Creation and Removal

```bestie
fun createDir(p: Path): void ! FsError            // fails if parent is missing
fun createDirAll(p: Path): void ! FsError          // creates intermediate dirs
fun remove(p: Path): void ! FsError                // file or empty dir
fun removeAll(p: Path): void ! FsError             // recursive — explicit, never implicit
```

Rules:

* Recursive removal is a **separate, explicitly named** function (`removeAll`) — never a flag
* `createDir` does not silently create parents; use `createDirAll` for that

### 9.2 Listing — `DirIterator`

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
own dir = readDir(Path.of("/var/log")) catch |err| { ... }
defer dir.close()

loop {
    val entry = dir.next() catch |err| { ... }
    match entry {
        End -> break
        path: Path -> process(path)
    }
}
```

Rules:

* `DirIterator` owns an OS handle and must be closed
* `next()` returns absent at end of directory and `! FsError` on a read failure — the two outcomes are genuinely different and both must be expressible. This is the one place in the standard library that needs `T ? ! E`, a form `core/types.md` §8.4 currently rejects; see the note there.
* Iteration order is **not** guaranteed across platforms
* Entries are returned as `Path`; call `stat` if metadata is needed

---

## 10. File Operations

```bestie
fun rename(from: Path, to: Path): void ! FsError       // atomic when same volume
fun copy(from: Path, to: Path): void ! FsError         // contents + permissions
fun symlink(target: Path, link: Path): void ! FsError
fun readLink(link: Path): Path ! FsError
```

Rules:

* `rename` is atomic on the same volume; cross-volume moves are an error, not a silent copy
* `copy` never follows into directories implicitly — it copies a single regular file
* Symlink support may be conditionally unavailable on some platforms (reported via `FsError`)

---

## 11. Error Model

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

## 12. Integration with Concurrency

`bestie.api.fs` introduces **no** async keywords. Blocking calls run on whatever thread invokes
them; concurrency is achieved with core `thread` or `fiber` from `bestie.lib.concurrency` (matching `bestie.api.io`):

```bestie
import bestie.lib.concurrency.fiber

fiber.of(() => {
    own stream = openRead(path) catch |err| { ... }
    defer stream.close()
    process(stream.read())
})
```

* File handles are **not** implicitly thread-safe
* A handle should be owned and used by one thread at a time

---

## 13. Relationship to Other APIs

* `bestie.api.fs` **builds on** `bestie.api.io` — it opens streams, it does not redefine them
* `bestie.api.fs` **uses** `bestie.api.os` for platform metadata and `bestie.lib.datetime` for timestamps
* `bestie.api.network` and `bestie.api.http` are independent peers, not built on `fs`

This preserves the clean layering described in `bestie.api.io`.

---

## 14. What `bestie.api.fs` Explicitly Excludes

* File watching / change notifications
* Glob / pattern-matching query language
* Memory-mapped files (→ `bestie.api.memory`)
* Temp-file lifecycle frameworks
* Implicit current-working-directory state
* Serialization or encoding of file contents

---

## 15. Stability and Evolution

* APIs are additive within a major version
* Platform-specific behavior is reported through `FsError`, never hidden
* No feature is added unless it cannot be expressed cleanly via streams + paths

---

## 16. Summary

`bestie.api.fs` is:

* Explicit — handles are owned and closed; paths are values
* Portable — one surface, honest about platform limits
* Composable — opens into `bestie.api.io` streams rather than reinventing I/O
* Predictable — typed errors, no exceptions, no hidden state

`File` and `DirIterator` are the only classes — the types that own live OS handles. `Path`,
`Permissions`, and `Metadata` are value/data snapshots. The file system is exposed as it is,
**without leaking OS chaos into the language**.

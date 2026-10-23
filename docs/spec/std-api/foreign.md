# Foreign Function Interface (FFI)

The purpose of this API is **interoperability**, not extension of the language semantics.
It allows Bestie programs to interoperate with **existing native codebases** (C, Assembly, and compatible ABIs) **without weakening Bestie’s safety guarantees**.

---

## 1. Design Goals

`bestie.api.foreign` exists to:

* Interoperate with existing systems and libraries
* Enable gradual migration to Bestie
* Support low-level system integration when required

While preserving:

* No-null guarantees
* Explicit ownership
* No hidden unsafe behavior
* Compile-time validation wherever possible

---

## 2. Non-Goals

This API intentionally does **not**:

* Introduce runtime reflection
* Allow unchecked pointer arithmetic
* Bypass ownership rules
* Enable dynamic language embeddings
* Support high-level foreign runtimes (e.g., JVM, Python VM)

Bestie remains a **native-first language**.

---

## 3. Supported Foreign Targets (Initial)

* **C (C99 ABI)**
* **Assembly (platform ABI)**
* Other languages only if they expose a stable C-compatible ABI

Examples:

* Rust (via `extern "C"`)
* Zig
* C++

---

## 4. Foreign Function Declarations

### 4.1 `foreign fun`

Foreign functions are declared explicitly:

```bestie
foreign fun strlen(s: ptr<const char>): int
```

Rules:

* No implementation body
* Signature must be fully explicit — every parameter is **named and typed**, exactly as in an ordinary `fun` declaration (`core/fp.md` §2.1). C header style, where a parameter may be a bare type, is not accepted
* A parameter the callee does not write is declared `ptr<const T>`, matching the C `const` qualifier (`core/memory.md` §8.4.1)
* ABI defaults to C unless specified
* Nullability must be expressed explicitly

---

### 4.2 ABI Specification

```bestie
foreign(c) fun memcpy(
    dest: ptr<byte>,
    src: ptr<byte>,
    size: int
): void
```

Supported ABI tags:

* `c`
* `system`
* `interrupt` and `naked` — definitions only (`core/freestanding.md` §5)
* Architecture-qualified conventions — `x86.sysv`, `x86.win64`, `arm.aapcs`, `riscv.lp64d`, …

C `union` types map to Bestie `union`, which always has C layout, and C bit-fields map to `bitfield` with the bit positions of the target ABI stated explicitly (`core/freestanding.md` §7).

### 4.3 Exporting to C — `export(c) fun`

The reverse direction: a Bestie function that C (or anything speaking the C ABI) can call. This is how a Bestie `staticlib` or `dynlib` target exposes its API (`core/modules-and-packaging.md` §13).

```bestie
@repr(C)
value class Version {
    major: int32
    minor: int32
}

export(c) fun acme_version(): Version {
    return Version.new(major: 1, minor: 4)
}

export(c) fun acme_checksum(data: ptr<const byte>, len: uint): uint32 {
    ...
}
```

Rules:

* The symbol name is the function name, **unmangled**. It must be unique across everything linked into the artifact, so the C convention of a library prefix (`acme_`) is the idiom.
* Every parameter and the return type must map to C (§5): primitives, `ptr<T>`, `ptr<T> ?`, non-capturing function pointers, and `@repr(C)` types. `str`, `slice<T>`, collections, error unions, and `own` classes cannot cross — pass a pointer and a length, and return status codes.
* An `export(c)` function cannot be generic, and must be `public`.
* **A panic never crosses the boundary.** C has no unwinding contract Bestie can rely on, so a panic that reaches an `export(c)` frame aborts the process. Convert recoverable failure to a status code before returning.
* The build writes a C header for the target — `include/<project>.h` — declaring every `export(c)` function and the `@repr(C)` types they use. The header is generated, never hand-maintained.

`export(c)` is the only way Bestie code gets a stable binary interface. Ordinary Bestie functions have none, by design (`core/modules-and-packaging.md` §13.2).

---

## 5. Type Mapping Rules

### 5.1 Primitive Mapping

| Bestie   | C         |
| -------- | --------- |
| `int`    | `intptr_t` |
| `int32`  | `int32_t`  |
| `int64`  | `int64_t`  |
| `uint`   | `uintptr_t` / `size_t` |
| `bool`   | `_Bool`    |
| `byte` / `uint8` | `uint8_t` |
| `char`   | `uint32_t` — a Unicode scalar, **not** a C `char`. A C `char*` is `ptr<byte>` (§5.3) |
| `ptr<T>` | `T*`       |

---

### 5.2 Struct Mapping

Bestie has no `struct` keyword — a C-ABI aggregate is a **`value class`** (`core/oop.md` §3.2): no identity, no vtable, copy-by-value, laid out inline.

Bestie types in ordinary code are **always packed by the compiler** (`core/memory-layout.md` §4). Declaration order is not the ABI.

To match a C header's declared layout and padding, mark the type `@repr(C)` — that is an FFI contract, not a core language mode. There is no `@layout(stable)` / `@stable` in core.

```bestie
@repr(C)
value class Point {
    x: int32
    y: int32
}
```

Under `@repr(C)` the compiler emits fields in **declaration order** with C's padding and alignment rules, suppressing the reordering it would otherwise perform. Prefer fixed-width types (`int32`, `uint64`) over `int` / `uint` in a `@repr(C)` type unless the C header genuinely uses `intptr_t` / `size_t`.

---

### 5.3 Strings

C strings are **NUL-terminated byte sequences**, so they map to `ptr<byte>` — never `ptr<char>`. Bestie's `char` is a 32-bit Unicode scalar (`core/types.md` §2.6), four times the width of a C `char`, and declaring a C string as `ptr<char>` would read every fourth byte.

```bestie
foreign fun strlen(s: ptr<byte>): uint
```

Conversion is explicit in both directions, because both directions have a cost and a failure mode:

```bestie
fun toCStr(s: str): own list<byte>           // appends a NUL; caller owns the buffer
fun fromCStr(p: ptr<byte>): own str ! ForeignError    // scans to NUL, validates UTF-8
```

Rules:

* `toCStr` allocates a NUL-terminated copy in a `list<byte>`; `c[0].address()` is the `ptr<byte>` to hand to C, and it converts to `ptr<const byte>` implicitly because adding `const` is always allowed (`core/memory.md` §8.8). A Bestie `str` is **not** NUL-terminated and may contain interior NUL bytes — one that does is `ForeignError.InteriorNul`, because truncating it silently is how injection bugs start.
* `fromCStr` is fallible: C makes no UTF-8 guarantee, and invalid bytes are `ForeignError.InvalidUtf8` rather than a replacement character.
* `fromCStr` **copies**. The returned `str` does not alias foreign memory, so it stays valid after the foreign buffer is freed.
* Neither function frees anything on the C side. If the C API says the caller must free the returned pointer, call its free function (§6).

---

## 6. Class Kinds and Ownership Rationale

| Type | Kind | Why |
| ---- | ---- | --- |
| A C-ABI aggregate | `value class` + `@repr(C)` | No identity, copy-by-value, laid out inline in declaration order (§5.2). |
| A C enum | `enum ... as int32` | Explicit discriminants pin the wire values (`core/oop.md` §3.3). |
| A foreign handle | `class` | An opaque pointer with a lifetime the C library defines. Wrapping it in a `class` with a `deinit` puts the release call next to the type. |
| `ForeignError` | `errors` | A closed set of boundary failures. |

Foreign memory has **no `own` semantics of its own** — the C library's documentation is the contract. Where a C API hands back memory the caller must free, wrap it in a `class` whose `deinit()` calls the C free function (`core/oop.md` §11.11), so the release is written once next to the type rather than at every call site.

---

## 7. Ownership and Safety Model

Foreign calls are **never assumed safe**.

Rules:

1. Ownership does **not** cross FFI boundaries implicitly
2. Returned pointers are treated as **unowned**
3. Caller must explicitly convert or wrap foreign memory
4. No automatic lifetime extension

```bestie
foreign fun malloc(size: uint): ptr<byte> ?
foreign fun free(p: ptr<byte>): void

// The C library owns the allocation; a wrapper puts the release next to the type.
class CBuffer {
    val ptr:  ptr<byte>
    val size: uint

    private init(p: ptr<byte>, n: uint) {
        this.ptr = p
        this.size = n
    }

    init(n: uint): ! ForeignError {
        val p = malloc(n) else { return ForeignError.AllocationFailed }
        this.init(p, n)
    }

    fun bytes(): slice<var byte>

    deinit() {
        free(this.ptr)
    }
}
```

```bestie
val own buf = try CBuffer.new(128)
defer buf.free()                  // runs deinit(), which calls C free()
```

`CBuffer.new` is fallible because `malloc` may return `NULL`, which the FFI layer surfaces as absent (§8).

---

## 8. No-Null Guarantee Preservation

Bestie has no `null`, no `nil`, and no nullable pointer type. These concepts do not exist in the language. A `ptr<T>` in Bestie code is always treated as a valid address — it is programmer responsibility to not construct an invalid one.

C APIs routinely return nullable pointers. At the FFI boundary, Bestie maps them to `ptr<T> ?`:

```bestie
// C: char* getenv(const char* name);  — may return NULL
foreign fun getenv(name: ptr<const byte>): ptr<byte> ?
```

The FFI layer performs the mapping automatically:
- C `NULL` (zero address) → absent
- Any non-zero C pointer → present

The caller handles the result like any other `T ?`:

```bestie
val own key = toCStr("PATH")
defer key.free()

if (val p = getenv(key[0].address())) {
    val own value = try fromCStr(p)
    defer value.free()
    use(value)
}
```

**`null` is not a keyword, a type, a value, or a literal in Bestie.** It cannot appear anywhere in Bestie source code. Any C function that may return `NULL` must be declared with `ptr<T> ?` as its return type — the compiler rejects a bare `ptr<T>` return for known-nullable C functions.

No implicit null propagation is possible because null does not exist to propagate.

---

## 9. Callbacks and Function Pointers

Callbacks are supported **without environment closures**. Only non-capturing callables are valid at the FFI boundary (see `core/fp.md` §7.3).

```bestie
foreign fun registerHandler(
    handler: fn(int) -> void
): void
```

The function-type spelling is core's (`core/fp.md` §6.1) — `fn(P) -> R`. There is no separate FFI syntax for it.

Rules:

* Callbacks must be non-capturing (named functions, local functions, or non-capturing lambdas)
* Fully static
* ABI-compatible
* No allocation
* Capturing lambdas (`[x]` / `[var x]`) are rejected at FFI registration sites

---

## 10. Error Handling

Foreign functions do not throw and do not return error unions. C reports failure in one of three ways, and each is translated **explicitly at the binding**, never automatically:

```bestie
errors ForeignError {
    AllocationFailed,
    InteriorNul,        // a str containing NUL cannot become a C string
    InvalidUtf8,        // a C string was not valid UTF-8
    NullReturn,         // a documented-non-null pointer came back null
    ErrorCode,          // the callee reported failure through its own convention
    LibraryNotFound,    // openLibrary could not load the file (§11.2)
    SymbolNotFound      // the library has no such symbol (§11.2)
}
```

**Sentinel return** — the C function returns a value that means failure:

```bestie
foreign fun open(path: ptr<const byte>, flags: int32): int32

fun openFile(path: str): int32 ! ForeignError {
    val own c = toCStr(path)
    defer c.free()

    val fd = open(c[0].address(), 0)
    if (fd < 0) { return ForeignError.ErrorCode }
    return fd
}
```

**Out parameter** — the C function writes the result through a pointer and returns a status:

```bestie
foreign fun parse(input: ptr<const byte>, out: ptr<int32>): int32

fun parseValue(s: str): int32 ! ForeignError {
    val own c = toCStr(s)
    defer c.free()

    var result: int32 = 0
    val status = parse(c[0].address(), result.address())
    if (status != 0) { return ForeignError.ErrorCode }
    return result
}
```

**Nullable return** — handled by `ptr<T> ?` (§8), with no manual check needed.

Bestie does **not** reinterpret foreign errors. A C `errno` is not translated into an `OsError`, and a library's error codes are not mapped onto a Bestie error set on your behalf — the binding author decides what each code means, because only they know.

---

## 11. Linking and Loading Native Libraries

### 11.1 Link-Time — Declared by the Project

A `foreign fun` declaration names a symbol, not a library. **Which library provides it is a build decision**, declared once in `bestie-project.toml` (`std-tools/package-manager.md` §4.5):

```toml
[native]
links = [
  { name = "ssl",    kind = "dynamic" },               # -lssl
  { name = "z",      kind = "static", search = "vendor/zlib/lib" },
  { name = "Cocoa",  kind = "framework", when = "target.os == 'macos'" },
]
```

* `kind` is `dynamic`, `static`, or `framework` (macOS). There is no default: the choice decides what must be present on the machine that runs the program, so it is written.
* `when` uses the same compile-time `target.*` constants as the language's `when` (`core/constants.md`).
* A dependency's `[native]` links propagate to the final artifact and are listed by `bestie tree --native`, so a program never links a system library nobody can see.

Keeping library names out of source means one `foreign fun` declaration serves every platform where the symbol lives in a differently named library.

### 11.2 Run-Time — Explicit Loading

Loading a library while the program runs — plugins, optional acceleration, a driver chosen by configuration — is explicit. It is never what `foreign fun` does.

```bestie
class Library {
    fun symbol<F>(name: str): ptr<F> ! ForeignError     // F is a fn(...) -> ... type
    fun close(): void
}

fun openLibrary(path: Path): own Library ! ForeignError
```

```bestie
val own lib = try openLibrary(Path.new("./plugins/libresize.so"))
defer lib.close()

val resize = try lib.symbol<fn(ptr<byte>, int32, int32) -> int32>("resize_image")
val status = resize.val(pixels, 640, 480)
```

Rules:

* `symbol<F>` trusts that the symbol has type `F` — a shared library carries no type information. It is the FFI boundary, with the same responsibility as a `foreign fun` declaration.
* A symbol pointer must not be called after `close()`. The library is an `own` value, so `close()` is visible, and the pointer is a raw `ptr`, so the rule is the programmer's — exactly as for any pointer into freed memory.
* Errors are `ForeignError.LibraryNotFound` and `ForeignError.SymbolNotFound`.
* A Bestie `dynlib` loaded this way is used through its `export(c)` functions only (§4.3). Two Bestie libraries in one process each contain their own copy of the standard library; no Bestie object, allocation, or panic crosses between them.

---

## 12. Platform-Specific Extensions

Platform-specific bindings must live under:

```text
bestie.api.foreign.<platform>
```

Examples:

```text
bestie.api.foreign.posix
bestie.api.foreign.win32
```

They must not alter core semantics.

---

## 13. Relationship to Other APIs

| API               | Responsibility            |
| ----------------- | ------------------------- |
| Core language     | Safety, ownership, ptr<T> |
| `bestie.api.memory`  | MMIO, memory regions      |
| `bestie.api.os`      | OS abstractions           |
| `bestie.api.foreign` | ABI interoperability      |

---

## 14. Intentional Restrictions

This API intentionally avoids:

* Automatic binding generation
* Runtime symbol lookup
* Dynamic loading by default
* Language-level unsafe blocks

Unsafe power is available — **only explicitly and locally**.

---

## 15. Summary

`bestie.api.foreign` is:

* Explicit
* Minimal
* ABI-focused
* Safety-preserving

It exists to **connect Bestie to the real world**, not to dilute its principles.

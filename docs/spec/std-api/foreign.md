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
foreign fun strlen(ptr<char>): int
```

Rules:

* No implementation body
* Signature must be fully explicit
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
* Platform-specific (namespaced)

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
| `char`   | `uint32_t` (Unicode scalar) |
| `ptr<T>` | `T*`       |

---

### 5.2 Struct Mapping

```bestie
struct Point {
    x: int
    y: int
}
```

* Layout is deterministic
* No padding surprises
* Must be explicitly marked if ABI-stable

```bestie
@repr(C)
struct Point { ... }
```

---

## 6. Ownership and Safety Model

Foreign calls are **never assumed safe**.

Rules:

1. Ownership does **not** cross FFI boundaries implicitly
2. Returned pointers are treated as **unowned**
3. Caller must explicitly convert or wrap foreign memory
4. No automatic lifetime extension

Example:

```bestie
foreign fun alloc(size: int): ptr<byte>

own buf = Memory.wrap(alloc(128), size = 128)
```

---

## 7. No-Null Guarantee Preservation

Bestie has no `null`, no `nil`, and no nullable pointer type. These concepts do not exist in the language. A `ptr<T>` in Bestie code is always treated as a valid address — it is programmer responsibility to not construct an invalid one.

C APIs routinely return nullable pointers. At the FFI boundary, Bestie maps them to `ptr<T> ?` (named `option<ptr<T>>` in std-lib):

```bestie
// C: char* getenv(const char* name);  — may return NULL
foreign fun getenv(name: ptr<char>): ptr<char> ?
```

The FFI layer performs the mapping automatically:
- C `NULL` (zero address) → absent
- Any non-zero C pointer → present

The caller handles the result like any other `T ?`:

```bestie
if (val p = getenv("PATH")) {
    usePathPtr(p)
}
```

**`null` is not a keyword, a type, a value, or a literal in Bestie.** It cannot appear anywhere in Bestie source code. Any C function that may return `NULL` must be declared with `ptr<T> ?` as its return type — the compiler rejects a bare `ptr<T>` return for known-nullable C functions.

No implicit null propagation is possible because null does not exist to propagate.

---

## 8. Callbacks and Function Pointers

Callbacks are supported **without environment closures**. Only non-capturing callables are valid at the FFI boundary (see `core/fp.md` §7.3).

```bestie
foreign fun registerHandler(
    handler: fun(int): void
): void
```

Rules:

* Callbacks must be non-capturing (named functions, local functions, or non-capturing lambdas)
* Fully static
* ABI-compatible
* No allocation
* Capturing lambdas (`[x]` / `[var x]`) are rejected at FFI registration sites

---

## 9. Error Handling

Foreign functions:

* Do not throw
* Do not return exceptions

Error handling patterns:

* Error codes
* Out parameters
* Explicit result types

Bestie does not reinterpret foreign errors.

---

## 10. Platform-Specific Extensions

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

## 11. Relationship to Other APIs

| API               | Responsibility            |
| ----------------- | ------------------------- |
| Core language     | Safety, ownership, ptr<T> |
| `bestie.api.memory`  | MMIO, memory regions      |
| `bestie.api.os`      | OS abstractions           |
| `bestie.api.foreign` | ABI interoperability      |

---

## 12. Intentional Restrictions

This API intentionally avoids:

* Automatic binding generation
* Runtime symbol lookup
* Dynamic loading by default
* Language-level unsafe blocks

Unsafe power is available — **only explicitly and locally**.

---

## 13. Summary

`bestie.api.foreign` is:

* Explicit
* Minimal
* ABI-focused
* Safety-preserving

It exists to **connect Bestie to the real world**, not to dilute its principles.

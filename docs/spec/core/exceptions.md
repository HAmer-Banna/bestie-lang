# Bestie — Error Handling

This document defines **how Bestie handles errors and failures**.

Bestie has two and only two failure paths:

* **`!` error returns** — for expected, recoverable failures. Compile-time typed values. Zero runtime cost.
* **Panics** — for invariant violations and unrecoverable faults. Terminate the program. Cannot be caught.

There is no `throw`, no exception hierarchy, and no runtime exception unwinding machinery.
`catch` exists only as an **inline operator on `!` values**, following Zig-style error handling. It is not an exception mechanism.

---

## 1. Design Goals

1. **Compile-time when possible** — all recoverable error types are known and checked at compile time.
2. **Explicit** — errors are visible at every call site. No hidden propagation.
3. **Zero cost** — errors are integer-tag values. No allocation, no stack walk, no unwinding.
4. **No surprises** — a panic terminates. An error return must be handled. Nothing is silent.

**Golden Rule:** If a failure can be represented as a typed value, it must use `!`. If a failure represents a violated invariant, it is a panic.

---

## 2. Error Returns (`!`)

`!` is the sole mechanism for expected, recoverable failures.

### 2.1 Error Sets

Errors are declared as lightweight named sets — not classes, no heap allocation:

```bestie
errors ParseError {
    InvalidFormat,
    Overflow,
    UnexpectedEnd
}

errors IoError {
    NotFound,
    PermissionDenied,
    Interrupted
}
```

* Tag-only, like enums
* Compile-time closed set
* Zero allocation — errors are integer tags
* Exhaustively matchable in `switch`

---

### 2.2 Function Signatures

`!` is postfix on the return type, mirroring `?`:

```bestie
fun parse(s: str): int ! ParseError         // returns int or ParseError
fun readFile(path: str): str ! IoError      // returns str or IoError
fun process(s: str): void !                  // error type inferred from body
```

A function returning `T ! E` either produces a `T` or an error value from set `E`.

---

### 2.3 Propagation with `try`

`try` unwraps the success value or immediately returns the error to the caller.
The caller must also declare `!` — propagation is always visible at the call site.

```bestie
fun run(): void ! {
    val n = try parse("123")   // propagates ParseError up if it fails
    print(n)
}
```

* No hidden propagation
* Every `try` is visible in source
* Compiler enforces the caller also declares `!`

---

### 2.4 Inline Recovery with `catch`

```bestie
val n = parse("123") catch 0

val n = parse("123") catch |e| switch (e) {
    case ParseError.InvalidFormat => -1
    case ParseError.Overflow      => -2
    case ParseError.UnexpectedEnd => -3
}
```

* `catch` handles only `!` error returns
* `catch` produces the fallback value inline
* Recovery remains explicit and local to the expression
* No allocation, no unwinding

---

### 2.5 Returning Errors Explicitly

```bestie
fun divide(a: int, b: int): int ! MathError {
    if (b == 0) {
        return MathError.DivisionByZero
    }
    return a / b
}
```

---

### 2.6 Exhaustive Matching

Error sets are closed and can be exhaustively matched in a `switch`:

```bestie
switch (err) {
    case ParseError.InvalidFormat => print("bad format")
    case ParseError.Overflow      => print("overflow")
    case ParseError.UnexpectedEnd => print("truncated")
}
```

The compiler enforces exhaustiveness — missing variants are compile-time errors.

---

## 3. Error Set Composition

Error sets are **composable at compile time**. Multiple error sets can be combined into a larger set — either named or anonymous. The compiler verifies composition at every `try` call site.

---

### 3.1 Named Error Set Union

A named error set can be declared as the union of existing sets:

```bestie
errors FileError = ParseError | IoError
```

`FileError` is a new flat error set whose variants are the union of all variants from `ParseError` and `IoError`. At runtime it is an integer tag — same representation as any other error set, just with more variants. The discriminant size follows the compact representation rules (§6.3 of `lang.md`).

Named unions can chain:

```bestie
errors AppError = FileError | NetworkError | AuthError
```

No nesting at runtime. `AppError` is always a flat set of integer tags.

---

### 3.2 Anonymous Union in Function Signatures

When a function can fail with errors from multiple sets, the union is written inline:

```bestie
fun process(path: str): int ! (ParseError | IoError)
```

Anonymous unions follow the same rules as named ones — flat, compact, exhaustively matchable. They are not a separate type from a named equivalent; `ParseError | IoError` and `FileError` (if declared identically) are structurally identical.

---

### 3.3 Subset Propagation — `try` Across Error Types

`try` works across error set boundaries when the callee's error type is a **subset** of the caller's error type. The compiler verifies this statically. No explicit conversion, no runtime cost — the error tag value is valid in the superset unchanged.

```bestie
errors FileError = ParseError | IoError

fun loadConfig(path: str): Config ! FileError {
    val raw = try readFile(path)      // readFile returns ! IoError  — IoError ⊆ FileError ✅
    val cfg = try parse(raw)          // parse returns ! ParseError — ParseError ⊆ FileError ✅
    return cfg
}
```

The compiler checks: is every variant in the callee's error set present in the caller's declared error set? If yes, `try` compiles to a direct propagation — one branch, zero overhead.

If the callee's error type is **not** a subset, the compiler rejects the bare `try` and requires explicit handling (see §3.4).

---

### 3.4 Explicit Error Mapping

When a callee's error type is not a subset of the caller's declared error type, the error must be explicitly converted at the call site:

```bestie
errors AppError { ConfigBad, NetworkDown, AuthFailed }

fun start(): void ! AppError {
    // IoError is not a subset of AppError — must map
    val cfg = try loadConfig("app.toml") catch |e| return AppError.ConfigBad

    // NetworkError is not a subset of AppError — map per variant
    try connect() catch |e| switch (e) {
        case NetworkError.Timeout    => return AppError.NetworkDown
        case NetworkError.Refused    => return AppError.NetworkDown
        case NetworkError.AuthFailed => return AppError.AuthFailed
    }
}
```

There is no implicit coercion between unrelated error sets. Every boundary crossing is visible in source.

---

### 3.5 Inferred Error Type (`!` without a set)

A function declared with bare `!` (no named set) has its error type **inferred by the compiler** from the body:

```bestie
fun run(): void ! {
    val n = try parse("123")      // contributes ParseError
    val f = try readFile("x.txt") // contributes IoError
    // inferred return type: void ! (ParseError | IoError)
}
```

The inferred type is the union of all error sets reachable via `try` in the body. It is resolved at compile time and written into the compiled signature — callers see the full concrete type.

Bare `!` is a convenience for functions where enumerating the full union upfront would be verbose. It does not weaken exhaustiveness or type safety — callers still see and handle the full set.

---

### 3.6 ABI — How `!` Returns Are Represented

A function returning `T ! E` uses a **register pair** on all targets:

| Register | Content on success | Content on error |
| -------- | ------------------ | ---------------- |
| Return register(s) | The `T` value | Undefined |
| Error register | `0` (no error) | Error tag (non-zero) |

On x86_64: `rax`/`xmm0` carries `T`, `rdx` carries the error tag.
On ARM64: `x0`/`v0` carries `T`, `x1` carries the error tag.

The error tag is a compact integer — its size follows §6.3 (minimum discriminant size). A 3-variant error set uses a `uint8` tag; the upper bytes of the error register are zero-extended.

For types `T` too large to fit in return registers, the caller passes a pointer to a result slot and the callee writes either `T` or the error tag into it via a discriminant flag at the slot's start.

**Zero overhead on the success path:** checking `rdx == 0` is a single comparison the branch predictor learns immediately. No allocation, no stack walk, no heap touch.

---

## 4. Panics — Unrecoverable Faults

A panic represents a **violated invariant**. The program cannot recover meaningfully. Panics terminate execution.

**Panics cannot be caught.** There is no mechanism to intercept them.

### 4.1 Sources of Panics

* Division by zero
* Stack overflow
* Memory allocation failure (when allocation is not fallible)
* Out-of-bounds index access (when not statically provable safe)
* Explicit `panic()` call
* Failed `assert()`

### 4.2 Behavior by Build Mode

| Build mode | Panic behavior                            |
| ---------- | ----------------------------------------- |
| `debug`    | Trap with message and source location     |
| `release`  | Terminate immediately (minimal overhead)  |
| `safe`     | Trap with message (release speed + checks)|

Build mode is set in `bestie-project.toml`.

### 4.3 Explicit Panic

```bestie
panic("unreachable state reached")
```

### 4.4 Assertions

```bestie
assert(x > 0)                          // panics if false
assert(x > 0, "x must be positive")   // panics with message
```

Assertions are active in `debug` and `safe` builds, elided in `release`.

---

## 4. Resource Cleanup — `defer`

`defer` replaces `finally` — and works for all scope exits, not just error paths.

See `lang.md` section 19 for the full `defer` specification.

```bestie
fun readConfig(path: str): str ! IoError {
    val f = try file.open(path)
    defer f.close()              // runs on every exit path
    return try f.readAll()
}
```

---

## 5. Integration with Core Return Types

| Mechanism | When to use | Layer |
| --------- | ----------- | ----- |
| `T ?` | Function return, parameter, or field that may be absent | **Core syntax** |
| `option<T>` | Named form: matching `Present` / `Not_Present`, helpers | **Std-lib** (`bestie.lib.utilities`) — same representation as `T ?` |
| `T ! E` | Recoverable failure with a typed reason — **the spelling for function signatures** | **Core syntax** |
| `result<T, E>` | Named form: matching `Ok` / `Err`, helpers | **Std-lib** (`bestie.lib.utilities`) — same representation as `T ! E` |
| `panic()` | Violated invariant — no recovery possible | Core |

`T ?` is presence. `T ! E` is success-or-error. They are not four systems. `option` / `result` are names for those two types so matching and helpers can evolve without touching core syntax.

Function signatures should use `T ?` and `T ! E`. Import `bestie.lib.utilities` when you need the names.

See `types.md` §8.3 and §8.4, and `std-lib/util.md`.

---

## 6. Compiler Enforcement

1. `!` error sets are **closed** — only declared variants are valid.
2. `try` requires the enclosing function to also declare `!`.
3. Unhandled `!` errors are **compile-time errors**.
4. Error set exhaustiveness in `switch` is **compile-time enforced**.
5. Panics are **not catchable** — no mechanism exists to intercept them.
6. `assert()` conditions are **compile-time evaluated when possible**.
7. `try` across error set boundaries is valid only when the callee's set is a **subset** of the caller's declared set — verified at compile time.
8. Bare `!` inferred types are resolved fully at compile time — the inferred type is concrete and visible in the compiled signature.
9. Named error set unions (`errors A = B | C`) are resolved at compile time to a flat set of variants — no runtime nesting.

---

## 7. What Bestie Deliberately Avoids

* `throw` and exception hierarchies
* Exception-style runtime catching
* Runtime exception unwinding machinery
* Unchecked exceptions — every `!` must be handled or propagated
* Hidden propagation — every `try` is visible at the call site
* Recoverable system faults modeled as hidden runtime traps

---

## 8. Summary

* `!` handles all recoverable failures — explicit, typed, zero-cost, compile-time checked.
* Panics handle all invariant violations — uncatchable, terminate.
* `defer` handles all cleanup — works on every exit path.

Nothing is hidden. Nothing propagates silently. Nothing allocates unexpectedly.

# Bestie — Error Handling

This document defines **how Bestie handles errors and failures**.

Bestie has two and only two failure mechanisms:

* **`!` error returns** — for expected, recoverable failures. Compile-time typed values. Zero runtime cost.
* **Panics** — for invariant violations and unrecoverable faults. Terminate the program. Cannot be caught.

There is no `throw`, no `catch`, no exception hierarchy, and no runtime exception unwinding machinery.

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

### 2.4 Inline Handling with `catch`

```bestie
val n = parse("123") catch |e| { 0 }                             // fallback value
val n = parse("123") catch |e ParseError.InvalidFormat| { -1 }   // typed match
```

* `catch` produces the fallback value inline
* Typed `catch` matches a specific error variant
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
    ParseError.InvalidFormat  => print("bad format")
    ParseError.Overflow       => print("overflow")
    ParseError.UnexpectedEnd  => print("truncated")
}
```

The compiler enforces exhaustiveness — missing variants are compile-time errors.

---

## 3. Panics — Unrecoverable Faults

A panic represents a **violated invariant**. The program cannot recover meaningfully. Panics terminate execution.

**Panics cannot be caught.** There is no mechanism to intercept them.

### 3.1 Sources of Panics

* Division by zero
* Stack overflow
* Memory allocation failure (when allocation is not fallible)
* Out-of-bounds index access (when not statically provable safe)
* Explicit `panic()` call
* Failed `assert()`

### 3.2 Behavior by Build Mode

| Build mode | Panic behavior                            |
| ---------- | ----------------------------------------- |
| `debug`    | Trap with message and source location     |
| `release`  | Terminate immediately (minimal overhead)  |
| `safe`     | Trap with message (release speed + checks)|

Build mode is set in `bestie-project.toml`.

### 3.3 Explicit Panic

```bestie
panic("unreachable state reached")
```

### 3.4 Assertions

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

| Mechanism            | When to use                                          |
| -------------------- | ---------------------------------------------------- |
| `?` partial function | Function may simply not return (no error reason)     |
| `! ErrorSet`         | Recoverable failure with a typed reason — preferred  |
| `Option<T>`          | Absence of a value (not a failure)                   |
| `Result<T, E>`       | Stdlib interop or composable error pipelines         |
| `panic()`            | Violated invariant — no recovery possible            |

---

## 6. Compiler Enforcement

1. `!` error sets are **closed** — only declared variants are valid.
2. `try` requires the enclosing function to also declare `!`.
3. Unhandled `!` errors are **compile-time errors**.
4. Error set exhaustiveness in `switch` is **compile-time enforced**.
5. Panics are **not catchable** — no mechanism exists to intercept them.
6. `assert()` conditions are **compile-time evaluated when possible**.

---

## 7. What Bestie Deliberately Avoids

* `throw` / `catch` — replaced by `!` and panics
* Exception hierarchies — replaced by error sets
* Runtime exception unwinding machinery
* Unchecked exceptions — every `!` must be handled or propagated
* Hidden propagation — every `try` is visible at the call site
* Recoverable system faults — violated invariants terminate, they do not throw

---

## 8. Summary

* `!` handles all recoverable failures — explicit, typed, zero-cost, compile-time checked.
* Panics handle all invariant violations — uncatchable, terminate.
* `defer` handles all cleanup — works on every exit path.

Nothing is hidden. Nothing propagates silently. Nothing allocates unexpectedly.

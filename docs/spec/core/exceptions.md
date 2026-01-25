# Bestie Exception Handling

This document defines **how Bestie handles errors and exceptions**.

Bestie is designed to be **safe, explicit, and predictable**. Unlike languages such as Java or Python, it avoids:

* Runtime surprises
* Unchecked exceptions
* Hidden null dereferences

All error handling in Bestie is **compile-time verified** and integrates cleanly with its memory and ownership model.

---

## 1. Design Goals

Bestie exception handling aims to:

1. **Be explicit** — programmers must decide how to handle errors.
2. **Be type-safe** — all error types are strongly typed and checked at compile time.
3. **Avoid hidden runtime costs** — exceptions do not implicitly allocate memory.
4. **Integrate with core return types** — works seamlessly with complete functions, partial functions (`?`), error returns (`!`), and `Option`.

**Golden Rule:** Exceptions in Bestie are for **rare, unrecoverable conditions**, not for normal control flow.

---

## 2. Exception Categories

Bestie distinguishes **two primary categories** of errors:

### 2.1 System Errors

* Represent runtime errors that **cannot be recovered from safely**.
* Thrown automatically by the runtime when invariants are violated.

Examples:

* `DivideByZeroError`
* `MemoryAllocationError`
* `StackOverflowError`

Example:

```bestie
fun divide(a: int, b: int): int {
    if b == 0 {
        throw DivideByZeroError("division by zero")
    }
    return a / b
}
```

### 2.2 Application Errors

* Represent **user-defined failures**.
* Can be defined as **classes extending `Error`**.
* Thrown explicitly by application code.

Example:

```bestie
class InvalidUserInputError : Error {
    val message: str

    fun new(msg: str) {
        message = msg
    }
}

throw InvalidUserInputError.new("Input must be positive")
```

---

## 3. Throwing Exceptions

* Use `throw` to raise an exception.
* Only objects derived from `Error` can be thrown.
* Throwing is **type-checked at compile time**.

```bestie
throw InvalidUserInputError.new("Invalid input")
```

---

## 4. Handling Exceptions

* Use `try/catch` blocks to handle exceptions.
* `catch` blocks are **ordered, type-checked, and compile-time verified**.

Example:

```bestie
try {
    val x = divide(10, 0)
} catch DivideByZeroError as e {
    print("Cannot divide by zero")
} catch Error as e {
    print("General error: ", e.message)
}
```

* Catch blocks may match **specific error types** or **general `Error` types**.

### 4.1 Finally Block

* Optional `finally` block executes **regardless of exception**.

```bestie
try {
    val f = file.open("config.txt")
} catch FileNotFoundError as e {
    print("File missing")
} finally {
    cleanup()
}
```

---

## 5. Integration with Core Return Types

Bestie exceptions work seamlessly with **core return types**:

| Return Type            | Behavior with Exceptions                             |
| ---------------------- | ---------------------------------------------------- |
| Complete function      | Can throw; caller may catch or propagate             |
| Partial function (`?`) | Must handle exceptions; compiler ensures correctness |
| Option class           | Exceptions coexist with `Not_Present` state          |
| Error return (`!`)     | Recoverable errors; alternative to `throw`           |

**Guideline:** Use `!` for expected recoverable errors and `throw` for unexpected, system-level failures.

---

## 6. Rules & Compiler Enforcement

1. Exceptions **do not cross thread boundaries** implicitly.
2. Only **`Error`-derived classes** may be thrown.
3. `throw` is **explicit**, and the compiler verifies all throw paths.
4. Runtime exceptions are **rare**; prefer partial functions, `!`, or `?` for expected failures.
5. Unhandled exceptions produce **compile-time warnings**.

---

## 7. Best Practices

* Prefer **error returns (`!`)** for normal, recoverable failures.
* Use **`throw`** for **unrecoverable or system-level errors**.
* Combine exceptions with **partial functions (`?`)** for clarity.
* Avoid mixing **`throw`** and `Option` without reason — stick to a single pattern per function.

---

## 8. Summary

* Bestie exceptions are **explicit, predictable, and safe**.
* Exceptions complement **return types, ownership rules, and memory safety**.
* They are **tools for rare, unrecoverable situations**, not general control flow.
* The compiler enforces rules **at compile time**, preventing accidental runtime surprises.

---

**Conclusion:** Bestie’s exception system supports **safe, high-performance system programming**, while remaining **expressive enough for backend logic**.


Do you want me to do that next?

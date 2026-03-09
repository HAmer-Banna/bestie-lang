# Bestie Language — Functional Programming (FP) & Functions Core Specification

This document defines **functional programming constructs**, **functions**, **lambdas**, and **function invocation semantics** in the Bestie core language.

Functional programming in Bestie is:

* Explicit
* Compile-time driven
* Allocation-aware
* Side-effect explicit
* Ownership-safe
* Fully interoperable with OOP and systems programming

FP in Bestie is **not a separate paradigm**.
It is a disciplined set of tools integrated directly into the core language.

Bestie is **multi-paradigm**, but **single-runtime**.

---

## 1. FP Design Philosophy

Bestie deliberately rejects:

* Hidden closures
* Implicit heap allocation
* Lazy evaluation by default
* Runtime-only abstractions
* Magical or inference-driven behavior

Bestie enforces:

* Explicit data flow
* Compile-time resolution
* Ownership-aware functions
* Deterministic execution
* Zero-cost abstractions

### Golden Rule

> If a function call, binding, capture, or dispatch can be resolved at compile time, it must be.

---

## 2. Functions

Functions are **first-class values**, but **not runtime objects by default**.

### 2.1 Function Declaration

```bestie
fun add(a: int, b: int): int {
    return a + b
}
```

Properties:

* Static dispatch by default
* No implicit captures
* No hidden allocation
* Explicit parameter and return types
* Resolved at compile time whenever possible

Functions may be:

* Top-level
* Class members (methods)
* Extension functions

---

### 2.2 Expression (Concise) Functions

Single-expression functions may omit braces:

```bestie
fun square(x: int): int = x * x
```

Rules:

* Single expression only
* No hidden allocation
* Inlined when possible

---

### 2.3 Default Parameters and Keyword Arguments

Functions and methods in Bestie may define **default parameter values** and may be called using **keyword arguments**.

```bestie
fun connect(host: str = "localhost", port: int = 5432, secure: bool = true): connection
```

Valid calls:

```bestie
connect()
connect("db.local")
connect(port = 5433)
connect(host = "db.local", secure = false)
```

Rules:

* Default values are compile-time constants or compile-time evaluable expressions
* Keyword arguments are resolved at compile time
* Argument reordering is allowed only when using keywords
* No runtime dispatch or allocation is introduced
* Defaults are applied at the call site during compilation

Default parameters and keyword arguments are **purely syntactic conveniences** and have **zero runtime cost**.

---

## 3. Complete vs Partial Functions

Bestie replaces null, nil, undefined, and sentinel values with a compile-time distinction between **complete** and **partial** functions.
There is no runtime empty value.

### 3.1 Complete Functions

A complete function guarantees that it returns a value on all execution paths.

```bestie
fun getUser(id: int): User {
    return repository.find(id)
}
```

Rules:

* All control-flow paths must return a value
* The compiler proves completeness
* Consumers do not need guards

Invalid:

```bestie
fun f(): int {
    if (cond) {
        return 1
    }
}
```

* Compile-time error: not all paths return a value.

### 3.2 Partial Functions

A partial function explicitly declares that it may not return a value.

```bestie
fun getUser(id: int): User ?
```

Rules:

* Caller must handle partiality
* Compiler enforces exhaustiveness
* No implicit exceptions

Example:

```bestie
fun getUser(id: int): User ? {
    if (exists(id)) {
        return repository.find(id)
    }
    return
}
```

#### 3.3 Calling Partial Functions

Calling a partial function forces the caller to handle control flow explicitly.

```bestie
fun process(id: int): void {
    val user = getUser(id)
    if (user) {
        sendEmail(user)
    }
}
```

Compiler enforces:

* Partial calls cannot be used as complete expressions
* Results must be guarded or transformed

### 3.4 Lambdas and Partiality

Lambdas may also be complete or partial.

Complete lambda:

```bestie
val inc = (x: int) => x + 1;
```

Partial lambda:

```bestie
val find = (x: int) => User ? {
    if (x > 0) return repo.get(x)
    return
}
```

---

## 4. Multiple Return Values and Tuples

Bestie allows functions to return **multiple values** using **tuples**.

### 4.1 Tuple Return Types

```bestie
fun divMod(x: int, y: int): (int, int) {
    return (x / y, x % y)
}
```

Rules:

* Tuples are value types
* Tuple layout is compile-time known
* No heap allocation is required
* Fully compatible with ownership and immutability rules

---

### 4.2 Tuple Return Shortcut

```bestie
fun stats(x: int): (int, int, int) {
    return x, x * 2, x * 3
}
```

Equivalent to:

```bestie
return (x, x * 2, x * 3)
```

Rules:

* Function return type must be a tuple
* Number and order of returned values must match the tuple type
* No runtime transformation is introduced

---

### 4.3 Tuple Destructuring (Capture Shortcut)

```bestie
val a, b = divMod(10, 3)
```

Equivalent to:

```bestie
val tmp = divMod(10, 3)
val a = tmp.0
val b = tmp.1
```

Rules:

* Destructuring is compile-time only
* No intermediate allocation is required
* Order is positional

---

### 4.4 Ignoring Values with `_`

```bestie
val quotient, _ = divMod(10, 3)
```

Rules:

* `_` introduces no binding
* Ignored values are not accessible
* Helps document intent and avoid unused-variable diagnostics

---

## 5. Function Types

Function types are **explicit and structural**.

```bestie
fn(int) -> int
```

Usage:

```bestie
val f: fn(int) -> int = square
```

Rules:

* No implicit boxing
* No hidden heap allocation
* Functions are referenced, not wrapped
* Fully known at compile time

---

## 6. Lambdas

Lambdas are anonymous functions with **explicit and restricted semantics**.

### 6.1 Lambda Syntax

```bestie
val f = (x: int): int => x * 2
```

Properties:

* Parameter types are explicit
* Return type inferred from body
* No implicit allocation
* Compile-time lowered

---

### 6.2 Non-Closure Rule (Core Guarantee)

**Lambdas in Bestie are not closures by default.**

By default, lambdas:

* Capture nothing
* Access only:

  * Their parameters
  * Compile-time constants
  * Global pure functions

Illegal:

```bestie
val y = 10
val f = (x: int) => x + y   // compile-time error
```

Guarantees:

* No hidden sharing
* No lifetime complexity
* Concurrency safety
* Zero allocation

---

### 6.3 Explicit Capture (Restricted)

```bestie
val factor: int = 3
val f = [factor](x: int) => x * factor
```

Rules:

* Captured values are copied
* Captures are immutable
* `own<T>` values cannot be captured
* Capture layout is compile-time known
* No heap allocation introduced

Preserves determinism while allowing controlled FP composition.

---

### 6.4 Lambda Allocation Model

By default:

* Lambdas do not allocate
* Lambdas are resolved at compile time
* Lowered to inline code or function pointers

Heap allocation for lambdas is **not part of the core language**.

---

## 7. Higher-Order Functions

```bestie
fun apply(f: fn(int) -> int, x: int): int {
    return f(x)
}
```

Rules:

* Function arguments are explicit
* No runtime boxing
* No dynamic dispatch unless explicitly annotated
* Fully compatible with ownership and concurrency rules

---

## 8. Variable Arguments (Varargs)

```bestie
fun sum(var xs: int...): int {
    var total = 0
    for x in xs {
        total += x
    }
    return total
}
```

Usage:

```bestie
sum(1, 2, 3, 4)
```

Rules:

* Varargs are explicit via `var`
* Element type must be specified
* Argument sequence is stack-allocated when possible
* No implicit heap allocation is introduced

---

## 9. Method References

### 9.1 Unbound Method References

```bestie
val f: fn(User) -> str = User::getName
```

Equivalent to:

```bestie
(u: User) => u.getName()
```

Rules:

* No instance capture
* Instance passed explicitly
* No allocation

---

### 9.2 Bound Method References (Restricted)

```bestie
own u = User.new()
val f: fn() -> str = u::getName
```

Rules:

* Bound object must be `own<T>` or a value type
* Ownership rules apply
* No reference capture
* Lowered at compile time

---

## 10. Extension Functions

Extension functions add behavior **without modifying types** and **without runtime cost**.

### 10.1 Declaration

```bestie
fun str.isEmpty(): bool {
    return this.length == 0
}
```

Usage:

```bestie
val s: str = "hello"
val empty = s.isEmpty()
```

---

### 10.2 Compilation Model

* Statically resolved
* Compiled as plain functions
* Desugared at compile time

```bestie
isEmpty(s)
```

No virtual dispatch, no vtables, no runtime lookup.

---

### 10.3 `this` Semantics

* `this` is the receiver parameter
* Immutable unless the type allows mutation
* Fully resolved at compile time

---

### 10.4 Extensions vs Members

Rules:

* Member functions always win
* No override is possible
* No polymorphism through extensions

Name collisions are illegal.

---

### 10.5 Extensions and Protocols

* Do not implement protocols
* Do not participate in protocol dispatch
* Are resolved statically at call site

---

### 10.6 Generic Extensions

```bestie
fun <T> list<T>.head(): T ? {
    if (this.size > 0) {
        return this[0]
    }
    return
}
```

Rules:

* Fully monomorphized
* No type erasure
* No runtime overhead

---

## 11. Function Composition

```bestie
fun compose<A, B, C>(
    f: (B) -> C,
    g: (A) -> B
): (A) -> C {
    return (x: A) => f(g(x))
}
```

No implicit currying or composition exists.

---

## 12. Currying and Partial Application

### 12.1 No Implicit Currying

Automatic currying is not supported.
Capturing-based currying is illegal.

### 12.2 Explicit Partial Application

Partial application is performed via **compile-time transformations**:

```bestie
val add5 = add.bind(5)
```

Rules:

* `bind` is compile-time only
* No allocation
* No captured state

---

## 13. Recursion

Recursion is explicit.

Rules:

* No guaranteed TCO
* Tail recursion may be optimized
* Stack usage is deterministic

---

## 14. FP and Memory Model

FP fully respects ownership.

Rules:

* `own<T>` cannot cross boundaries implicitly
* Ownership passing is explicit
* Lambdas cannot capture owning references

Ensures:

* Manual memory safety
* Concurrency correctness

---

## 15. FP and OOP Interoperability

* Methods are functions with receivers
* Extension functions bridge FP and OOP
* Protocols define behavior contracts
* Lambdas replace many OO patterns

FP reduces accidental complexity.
It does not replace OOP.

---

## 16. Immutability in FP

FP in Bestie strongly prefers immutability.

Guidelines:

* Use `val` by default
* Prefer value types
* Avoid `var` in FP-heavy code
* Favor transformation over mutation

```bestie
val users2 = std.functional.map(users, u => u.withName("Alice"))
```

Enforced by:

* Type system
* Ownership rules
* Compile-time checks

---

## 17. What Bestie Deliberately Avoids in FP

* Implicit currying
* Lazy evaluation by default
* Runtime monads
* Hidden effect systems
* Reflection-based dispatch
* Closure-heavy abstractions

---

## 18. Summary

Functional programming in Bestie is:

* Explicit
* Compile-time resolvable
* Allocation-aware
* Ownership-safe
* Concurrency-safe
* Zero-cost by design

FP in Bestie exists to **compose behavior clearly**, not to obscure execution.

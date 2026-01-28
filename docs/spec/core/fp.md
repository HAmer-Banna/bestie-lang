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

## 3. Multiple Return Values and Tuples

Bestie allows functions to return **multiple values** using **tuples**.

### 3.1 Tuple Return Types

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

### 3.2 Tuple Return Shortcut

As a convenience, tuple construction may be omitted in `return` statements.

```bestie
fun stats(x: int): (int, int, int) {
    return x, x * 2, x * 3
}
```

This is **exactly equivalent** to:

```bestie
return (x, x * 2, x * 3)
```

Rules:

* The function return type must be a tuple
* The number and order of returned values must match the tuple type
* No runtime transformation is introduced

---

### 3.3 Tuple Destructuring (Capture Shortcut)

Tuple values may be destructured directly at the binding site.

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

### 3.4 Ignoring Values with `_`

Unused tuple values may be ignored using `_`.

```bestie
val quotient, _ = divMod(10, 3)
```

Rules:

* `_` introduces no binding
* Ignored values are not accessible
* Helps document intent and avoid unused-variable diagnostics

---

## 4. Function Types

Function types are **explicit and structural**.

```bestie
(int) -> int
```

Usage:

```bestie
val f: (int) -> int = square
```

Rules:

* No implicit boxing
* No hidden heap allocation
* Functions are referenced, not wrapped
* Fully known at compile time

---

## 5. Lambdas

Lambdas are anonymous functions with **explicit and restricted semantics**.

### 5.1 Lambda Syntax

```bestie
val f = (x: int): int => x * 2
```

Properties:

* Parameter types are explicit
* Return type inferred from body
* No implicit allocation
* Compile-time lowered

---

### 5.2 Non-Closure Rule (Core Guarantee)

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

This guarantees:

* No hidden sharing
* No lifetime complexity
* Concurrency safety
* Zero allocation

---

### 5.3 Explicit Capture (Restricted)

Explicit capture is allowed with strict rules.

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

This preserves determinism while allowing controlled FP composition.

---

### 5.4 Lambda Allocation Model

By default:

* Lambdas do not allocate
* Lambdas are resolved at compile time
* Lowered to inline code or function pointers

Heap allocation for lambdas is **not part of the core language**.

---

## 6. Higher-Order Functions

Bestie fully supports higher-order functions.

```bestie
fun apply(f: (int) -> int, x: int): int {
    return f(x)
}
```

Rules:

* Function arguments are explicit
* No runtime boxing
* No dynamic dispatch unless explicitly annotated
* Fully compatible with ownership and concurrency rules

---

## 7. Variable Arguments (Varargs)

Bestie supports **variable-length argument lists** using explicit vararg parameters.

```bestie
fun sum(var xs: int): int {
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

## 8. Method References

Bestie supports explicit, safe method references.

### 8.1 Unbound Method References

```bestie
val f: (User) -> str = User::getName
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

### 8.2 Bound Method References (Restricted)

```bestie
own u = User.init(...).new()
val f: () -> str = u::getName
```

Rules:

* Bound object must be `own<T>` or a value type
* Ownership rules apply
* No reference capture
* Lowered at compile time

---

## 9. Partial Functions and Errors

Bestie avoids nulls and implicit exceptions.

### 9.1 Partial Functions

```bestie
fun parseInt(s: str): int?
```

Rules:

* Partiality is explicit
* Caller must handle it
* Compiler enforces exhaustiveness

---

### 9.2 Option and Error-Oriented FP

Preferred FP-style returns:

* `option<T>`
* `T?`
* Explicit error values

```bestie
fun findUser(id: int): option<User>
```

Errors are values.
Control flow is explicit.

---

## 10. Immutability in FP

FP in Bestie strongly prefers immutability.

Guidelines:

* Use `val` by default
* Prefer value types
* Avoid `var` in FP-heavy code
* Favor transformation over mutation

```bestie
val users2 = users.map(u => u.withName("Alice"))
```

Immutability is enforced by:

* Type system
* Ownership rules
* Compile-time checks

---

## 11. Extension Functions

Extension functions add behavior **without modifying types** and **without runtime cost**.

### 11.1 Declaration

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

### 11.2 Compilation Model

Extension functions are:

* Statically resolved
* Compiled as plain functions
* Desugared at compile time

```bestie
isEmpty(s)
```

There is:

* No virtual dispatch
* No vtables
* No runtime lookup

---

### 11.3 `this` Semantics

Inside an extension:

* `this` is the receiver parameter
* Immutable unless the type allows mutation
* Fully resolved at compile time

---

### 11.4 Extensions vs Members

Rules:

* Member functions always win
* No override is possible
* No polymorphism through extensions

Name collisions are illegal.

---

### 11.5 Extensions and Protocols

Extensions:

* Do not implement protocols
* Do not participate in protocol dispatch
* Are resolved statically at call site

---

### 11.6 Generic Extensions

```bestie
fun <T> list<T>.head(): T? {
    return if (this.size > 0) this[0]
}
```

Rules:

* Fully monomorphized
* No type erasure
* No runtime overhead

---

## 12. Function Composition

Composition is explicit and type-safe.

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

## 13. Currying and Partial Application

### 13.1 No Implicit Currying

Automatic currying is not supported.

Capturing-based currying is illegal.

---

### 13.2 Explicit Partial Application

Partial application is performed via **compile-time transformations**:

```bestie
val add5 = add.bind(5)
```

Rules:

* `bind` is compile-time only
* No allocation
* No captured state

---

## 14. Recursion

Recursion is explicit.

Rules:

* No guaranteed TCO
* Tail recursion may be optimized
* Stack usage is deterministic

---

## 15. FP and Memory Model

FP fully respects ownership.

Rules:

* `own<T>` cannot cross boundaries implicitly
* Ownership passing is explicit
* Lambdas cannot capture owning references

This ensures:

* Manual memory safety
* Concurrency correctness

---

## 16. FP and OOP Interoperability

* Methods are functions with receivers
* Extension functions bridge FP and OOP
* Protocols define behavior contracts
* Lambdas replace many OO patterns

FP reduces accidental complexity.
It does not replace OOP.

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

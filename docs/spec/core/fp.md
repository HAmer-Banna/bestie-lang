# Bestie Language — Functional Programming (FP) & Functions Core Specification

This document defines **functional programming constructs**, **functions**, and **lambdas** in the Bestie core language.

Functional programming in Bestie is:

* Explicit
* Compile-time driven
* Allocation-aware
* Side-effect explicit
* Ownership-safe
* Fully interoperable with OOP and systems programming

FP in Bestie is **not a separate paradigm**.
It is a set of disciplined tools integrated into the core language.

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
* Class members
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

## 3. Function Types

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

## 4. Lambdas

Lambdas are anonymous functions with **explicit and restricted semantics**.

### 4.1 Lambda Syntax

```bestie
val f = (x: int): int => x * 2
```

Properties:

* Parameter types are explicit
* Return type inferred from body
* No implicit allocation
* Compile-time lowered

---

### 4.2 Non-Closure Rule (Core Guarantee)

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

### 4.3 Explicit Capture (Restricted)

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

### 4.4 Lambda Allocation Model

By default:

* Lambdas do not allocate
* Lambdas are resolved at compile time
* Lowered to inline code or function pointers

Heap allocation for lambdas is **not part of the core language**.

---

## 5. Higher-Order Functions

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

## 6. Method References

Bestie supports explicit, safe method references.

### 6.1 Unbound Method References

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

### 6.2 Bound Method References (Restricted)

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

## 7. Partial Functions and Errors

Bestie avoids nulls and implicit exceptions.

### 7.1 Partial Functions

```bestie
fun parseInt(s: str): int?
```

Rules:

* Partiality is explicit
* Caller must handle it
* Compiler enforces exhaustiveness

---

### 7.2 Option and Error-Oriented FP

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

## 8. Immutability in FP

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

## 9. Extension Functions

Extension functions add behavior **without modifying types** and **without runtime cost**.

### 9.1 Declaration

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

### 9.2 Compilation Model

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

### 9.3 `this` Semantics

Inside an extension:

* `this` is the receiver parameter
* Immutable unless the type allows mutation
* Fully resolved at compile time

---

### 9.4 Extensions vs Members

Rules:

* Member functions always win
* No override is possible
* No polymorphism through extensions

Name collisions are illegal.

---

### 9.5 Extensions and Protocols

Extensions:

* Do not implement protocols
* Do not participate in protocol dispatch
* Are resolved statically at call site

---

### 9.6 Generic Extensions

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

## 10. Function Composition

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

## 11. Currying and Partial Application

### 11.1 No Implicit Currying

Automatic currying is not supported.

Capturing-based currying is illegal.

---

### 11.2 Explicit Partial Application

Partial application is performed via **compile-time transformations**:

```bestie
val add5 = add.bind(5)
```

Rules:

* `bind` is compile-time only
* No allocation
* No captured state

---

## 12. Recursion

Recursion is explicit.

Rules:

* No guaranteed TCO
* Tail recursion may be optimized
* Stack usage is deterministic

---

## 13. FP and Memory Model

FP fully respects ownership.

Rules:

* `own<T>` cannot cross boundaries implicitly
* Ownership passing is explicit
* Lambdas cannot capture owning references

This ensures:

* Manual memory safety
* Concurrency correctness

---

## 14. FP and OOP Interoperability

* Methods are functions with receivers
* Extension functions bridge FP and OOP
* Protocols define behavior contracts
* Lambdas replace many OO patterns

FP reduces accidental complexity.
It does not replace OOP.

---

## 15. What Bestie Deliberately Avoids in FP

* Implicit currying
* Lazy evaluation by default
* Runtime monads
* Hidden effect systems
* Reflection-based dispatch
* Closure-heavy abstractions

---

## 16. Summary

Functional programming in Bestie is:

* Explicit
* Compile-time resolvable
* Allocation-aware
* Ownership-safe
* Concurrency-safe
* Zero-cost by design

FP in Bestie exists to **compose behavior clearly**, not to obscure execution.


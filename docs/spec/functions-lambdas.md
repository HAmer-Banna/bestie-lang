# Functions & Lambdas — Core Language Specification

This document defines **functions**, **lambdas**, and **functional constructs** in the Bestie core language.

The goal is to support **modern functional programming techniques** while preserving:

* Compile-time determinism
* Zero hidden allocation
* Explicit data flow
* Simplicity and predictability

Bestie is **multi-paradigm**, but not multi-runtime.

---

## 1. Design Principles

Functions and lambdas in Bestie follow these principles:

1. **Functions are first-class**
2. **No closures**
3. **No implicit state capture**
4. **Compile-time resolution by default**
5. **No runtime function objects unless explicitly requested**
6. **Readable over clever**
7. **One way to express each idea**

---

## 2. Functions

### 2.1 Function Declaration

```bestie
fun add(a: int, b: int): int {
    return a + b
}
```

Properties:

* Top-level or inside classes
* May return `void`
* May be `pub`, `pkg`, `protec`, `priv`
* Resolved at compile time

---

### 2.2 Concise Functions

```bestie
fun inc(x: int): int = x + 1
```

Rules:

* Single-expression body
* No hidden allocation
* Inlined when possible

---

## 3. Function Types

Function types are explicit and structural.

```bestie
(int, int) -> int
```

Usage:

```bestie
val f: (int) -> int = inc
```

Rules:

* No implicit boxing
* No hidden heap allocation
* Functions are referenced, not wrapped

---

## 4. Lambdas

### 4.1 Lambda Syntax

```bestie
val f = { x: int -> x + 1 }
```

Lambda syntax is deliberately minimal.

---

### 4.2 Non-Closure Rule (Critical)

**Lambdas in Bestie are NOT closures.**

They:

* Cannot capture variables from outer scopes
* Can only access:

  * Their parameters
  * Compile-time constants
  * Global pure functions

Illegal:

```bestie
val y = 10
val f = { x: int -> x + y }   // compile-time error
```

Legal:

```bestie
val f = { x: int -> x + 1 }
```

This rule guarantees:

* No hidden sharing
* No lifetime complexity
* Safe use in concurrency
* Zero allocation

---

### 4.3 Lambda Allocation

By default:

* Lambdas do **not** allocate
* Lambdas are resolved at compile time
* Lambdas are lowered to function pointers or inline code

Heap allocation only occurs if explicitly requested (future extension, not core).

---

## 5. Lambdas and Concurrency

Lambdas used in `threadOS` or `threadLight`:

* Must obey ownership rules
* Cannot capture state
* Are safe by construction

Example:

```bestie
threadLight.start { 
    process(job)
}
```

This is safe because:

* `job` must be passed explicitly
* Ownership rules are enforced

---

## 6. Method References

Bestie supports **method references**, explicitly and safely.

### 6.1 Instance Method Reference

```bestie
val f: (User) -> str = User::getName
```

Meaning:

* No instance captured
* Instance is passed explicitly as parameter

Equivalent to:

```bestie
{ u: User -> u.getName() }
```

---

### 6.2 Bound Method References (Restricted)

Bound references are allowed **only with value types or owned values**.

```bestie
own u = User.init(...).new()
val f: () -> str = u::getName
```

Rules:

* Bound object must be `own` or value
* Ownership rules apply
* No reference capture

---

## 7. Currying and Partial Application

### 7.1 Currying (Explicit Only)

Bestie does **not** support automatic currying.

Instead, currying is explicit and readable:

```bestie
fun add(a: int): (int) -> int {
    return { b: int -> a + b }   // ❌ illegal (captures a)
}
```

This is illegal because it captures `a`.

Correct approach:

```bestie
fun add(a: int, b: int): int = a + b

fun add5(b: int): int = add(5, b)
```

**Rationale**:

* No hidden closures
* No heap allocation
* No lifetime complexity

---

### 7.2 Partial Application via Method References

This is the **preferred** idiom:

```bestie
fun add(a: int, b: int): int

val add5 = add.bind(5)
```

Rules:

* `bind` is a compile-time transformation
* Produces a new function with reduced arity
* No allocation
* No captured state

---

## 8. Higher-Order Functions

Bestie fully supports higher-order functions.

Example:

```bestie
fun apply(x: int, f: (int) -> int): int {
    return f(x)
}
```

Rules:

* Function arguments are explicit
* No implicit lambdas
* No inference-driven ambiguity

---

## 9. Functional Utilities (Core vs std-lib)

### 9.1 Core Language

Core provides:

* Function types
* Lambdas
* Method references

---

### 9.2 std-lib Functional Utilities

std-lib provides:

* `map`
* `filter`
* `fold`
* `zip`

All are:

* Free functions
* Allocation-explicit
* Deterministic
* Non-magical

---

## 10. No Implicit Functional Magic

Bestie explicitly does **not** support:

* Closures
* Lazy evaluation
* Implicit currying
* Implicit monads
* Implicit async
* Implicit effect systems

These belong to languages with different goals.

---

## 11. Why This Design Works

This model:

* Supports FP patterns cleanly
* Avoids Scala-level complexity
* Keeps compilation fast
* Keeps mental model small
* Integrates perfectly with ownership and concurrency
* Is safe by default

Functional programming in Bestie is **explicit, disciplined, and boring** — which is exactly what large systems need.

---

## 12. Summary

Functions and lambdas in Bestie are:

* First-class
* Compile-time resolved
* Non-capturing
* Allocation-free
* Concurrency-safe
* Readable and predictable

Bestie supports functional programming **as a tool**, not as a religion.

---

**This finalizes function and lambda semantics in the Bestie core language.**

# Bestie Language — Core Specification

## 1. Overview

Bestie is a native, compiled programming language designed from first principles for **systems programming and backend engineering**.

Bestie is not built around a single paradigm. Object‑oriented programming, functional programming, and low‑level control are treated as **explicit tools**, not ideologies.

Bestie prioritizes:

* Compile‑time resolution
* Deterministic execution
* Predictable memory layout
* Zero‑cost abstractions
* Long‑term stability

Bestie is conceptually:
**Zig + Kotlin**, without inheriting the weaknesses of either.

---

## 2. Core Design Principles

The Bestie compiler resolves **everything that is resolvable** at compile time, including:

* Type inference
* Generic specialization (monomorphization)
* Memory layout, padding, and alignment
* Protocol dispatch
* Inline expansion
* Destructor placement
* Ownership validation
* Pointer safety
* Builder chains
* Error‑handling paths
* Loop lowering and unrolling opportunities
* Devirtualization
* Concurrency safety (races, deadlocks, sharing rules)

The runtime is intentionally minimal.

**Golden Rule**

> If something can be resolved at compile time, it must be.

---

## 3. Language Structure

A Bestie program consists of:

* Packages
* Modules
* Types
* Functions
* Protocols
* Annotations

The core language is **small and sealed**.

Higher‑level abstractions live in:

* `std-lib`
* `std-api`
* `std-framework`

The core defines guarantees. APIs define convenience.

---

## 4. Variables and Bindings

Bestie provides three binding forms.

### 4.1 `const` — Compile‑Time Constant

`const` defines an immutable binding and immutable value resolved entirely at compile time.

Properties:

* Immutable binding and value
* No runtime allocation
* No mutation through references
* Cannot reference mutable memory

Valid scopes:

* File
* Class
* Protocol

Invalid scopes:

* Function parameters
* Runtime locals

---

### 4.2 `val` — Immutable Binding (Default)

`val` defines an immutable binding to a value that may be mutable depending on its type.

Properties:

* Runtime values allowed
* Preferred default
* Encourages immutability

File‑level `val` must be annotated with `@immutable`.

---

### 4.3 `var` — Mutable Binding (Restricted)

`var` defines a mutable binding and is deliberately restricted.

Allowed only:

* Local variables
* Class properties with explicit getters/setters

Disallowed:

* File scope
* Protocols
* Data/value/single classes

---

## 5. Types

### 5.1 Primitive Types

Primitive value types map directly to machine types:

* `byte`
* `int32`, `int64`
* `uint32`, `uint64`
* `float32`, `float64`
* `int`, `uint`, `float` (width inferred by target)
* `bool`
* `char`

All primitives:

* Have no headers
* Are stack‑friendly
* Require no deallocation

---

### 5.2 Core Value Types

Value types include:

* `str` (UTF‑8, immutable)
* `tuple`
* `ptr<T>`
* Immutable collection literals

Value types:

* Are immutable by default
* Are copied efficiently
* Have no hidden allocation

---

## 6. Control Flow Expressions

### 6.1 `if` as Statement and Expression

In Bestie, `if` is both a **control statement** and an **expression**.

#### Expression Form

When used as a value, `if` **must produce a value**:

```bestie
val x: int = if (cond) 4 else 0
```

For primitive defaults:

* `int` → `0`
* `float` → `0.0`
* `bool` → `false`
* `char` → `''`
* `str` → `""`

If no natural empty value exists, an `option<T>` must be used.

---

#### Partial Function Form

Inside functions, `if` may omit `else`. In that case the function becomes **partial**:

```bestie
fun check(x: int): bool? =
    if (x > 0) true
```

Partial functions and methods are defined formally in **fp.md**.

---

### 6.2 `switch` as Statement and Expression

`switch` is both a **control statement** and an **expression**. and **has no fallthrough**.

Properties:

* Compact syntax
* Exhaustiveness enforced when used as an expression
* No implicit control flow

---

## 7. Loops

Bestie supports:

* `for`
* `for in`
* `while`

### 7.1 Loop `else`

Loops may include an `else` clause:

* Executed only if the loop completes normally
* Skipped if the loop exits via `break`

---

### 7.2 Loops as Expressions

Loops may be **assigned to values or collections** when resolvable at compile time.

```bestie
val x: int = for (i = 0; i < 5; i++) i + 5

val xs: list<int> = for (i = 0; i < 5; i++) i * 2
```

This is conceptually similar to streams or comprehensions, but:

* Fully compile‑time lowered
* No iterator allocation
* No runtime abstraction cost

---

## 8. Syntax Rules

### 8.1 Parentheses and Braces

* `()` are **always required** after `if`, `for`, `while`
* `{}` are **required** when the body is not on the same line

Valid:

```bestie
if (cond) doSomething() else doOther()
```

Invalid:

```bestie
if (cond)
  doSomething()   // compiler error
```

---

### 8.2 Brace Placement

Bestie does **not** require `{` to be on the same line (unlike Go).

Formatting is standardized by the `bestie fmt` tool.

---

## 9. Operators

### 9.1 Logical Operators

* `&&`, `||`, `!`
* Short‑circuit semantics

### 9.2 Bitwise Operators

* `&`, `|`, `^`, `~`
* `<<`, `>>`

### 9.3 Type & Introspection Operators

* `is` — type check (compile‑time when possible)
* `typeOf(T)` — compile‑time type token
* `sizeOf(T)` — compile‑time size in bytes

---

## 10. Words vs Symbols

Bestie intentionally balances **symbols and keywords**.

Examples:

* Uses symbols: `&&`, `||`, `==`
* Uses words: `ext`, `impl`

Bestie avoids:

* Python‑style fully word‑based logic (`and`, `or`, `not`)
* Symbol‑heavy DSL‑like syntax

The goal is **clarity without ceremony**.

---

## 11. Equality and Identity

### 11.1 `==` — Structural Equality

* Value‑based comparison
* Lowered at compile time
* Uses `Equable.equal` when available
* Otherwise compiler‑generated for value types

### 11.2 `is` — Identity / Type Relation

* For value types: structural identity
* For reference types: identity comparison
* For types: subtype relation

No runtime reflection is involved.

---

## 12. What Is Intentionally Excluded

Bestie deliberately excludes:

* Garbage collection
* Reflection
* Macros
* Unsafe blocks
* Runtime metaprogramming
* Implicit dynamic dispatch
* Global mutable state

---

## 13. Stability & Versioning

* Core language is sealed
* Backward compatibility is mandatory
* Experimental features require compiler flags
* APIs evolve without breaking core guarantees

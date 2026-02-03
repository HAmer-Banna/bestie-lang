# Bestie Language — Core Specification

## 1. Overview

Bestie is a native, compiled programming language designed from first principles for **systems programming and backend engineering**.

Bestie is not built around a single paradigm.  
Object-oriented programming, functional programming, and low-level control are treated as **explicit tools**, not ideologies.

Bestie prioritizes:

* Compile-time resolution
* Deterministic execution
* Predictable memory layout
* Zero-cost abstractions
* Long-term stability

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
* Error handling paths
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

Higher-level abstractions live in:

* `std-lib`
* `std-api`
* `std-framework`

The core defines guarantees.  
APIs define convenience.

---

## 4. Variables and Bindings

Bestie provides three binding forms.

### 4.1 `const` — Compile-Time Constant

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

Intended use:

* Mathematical constants
* Compile-time configuration
* Type-level invariants

---

### 4.2 `val` — Immutable Binding (Default)

`val` defines an immutable binding to a value that may be mutable depending on its type.

Properties:

* Runtime values allowed
* Preferred default
* Encourages immutability

File-level `val` must be annotated with `@immutable`.

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

### 4.4 Multiple Value Declarations

Bestie makes a **strict distinction** between *multiple value declarations* and *tuples*.

Multiple value declarations are a **binding convenience**, not a data structure.

```bestie
val x, y, z = 5, 6, 3
````

This is equivalent to:

```bestie
val x: int = 5
val y: int = 6
val z: int = 3
```

Rules:

* All values on the right-hand side **must have the same type**
* The common type may be inferred or explicitly annotated
* Heterogeneous values are **not allowed**
* No tuple value is created
* Binding is positional and resolved at compile time

Explicit typing is allowed:

```bestie
val x, y, z: int = 5, 6, 3
```

Invalid example (heterogeneous values):

```bestie
val x, y, z = 5, 6, 'c'    // ❌ illegal
```

Binding form restrictions:

* All bindings in a declaration **must use the same keyword**
* Mixing `val`, `var`, and `const` in a single declaration is illegal

Invalid examples:

```bestie
val x, var y = 1, 2        // ❌ illegal
const x, y = 1, 2         // ❌ illegal (runtime values)
```

Valid examples:

```bestie
val a, b = getPairInts()
var i, j = 0, 1
```

Multiple value declarations are **pure syntax sugar** and are lowered by the compiler into independent bindings with no runtime cost.

---

### 4.5 Tuples and Destructuring Bindings

Tuples are **first-class value types** that may contain **heterogeneous values**.

```bestie
val t: tuple = (5, 6, 'c')
```

Tuple destructuring creates bindings from a tuple value:

```bestie
val (x, y, z) = (5, 6, 'c')
```

Or with explicit type:

```bestie
val (x, y, z): tuple = (5, 6, 'c')
```

Rules:

* Tuples may contain values of different types
* Destructuring requires a tuple value
* Parentheses **explicitly signal tuple semantics**
* Tuple layout and arity are compile-time known

This distinction ensures that **binding convenience never replaces data modeling**.

---

### 4.6 Ignoring Values with `_`

Bestie allows the underscore identifier `_` to explicitly **ignore values** wherever a binding would otherwise be required.

```bestie
val x, _, z = 1, 2, 3
val (_, y, _) = 5, 6, 'c'
```

Rules:

* `_` introduces **no binding** and no lifetime
* `_` may appear in any position
* `_` is allowed in multiple declarations and tuple destructuring
* `_` is allowed wherever ignoring a value does not violate Bestie’s type, ownership, or mutability rules
* `_` suppresses unused-value diagnostics

Restrictions:

* `_` cannot be read from or assigned to
* `_` is not a valid identifier

All `_` usage is **compile-time only** and introduces no runtime behavior.

---

## 5. Types

### 5.1 Primitive Types

Primitive value types map directly to machine types:

* `byte`
* `ubyte`
* `int16`, `int32`, `int64`
* `uint16`, `uint32`, `uint64`
* `float32`, `float64`
* `int`, `uint`, `float` (width inferred by target)
* `bool`
* `char`
* `void`

All primitives:

* Have no headers
* Are stack-friendly
* Require no deallocation

---

### 5.2 Core Value Types

Value types include:

* `str` (UTF-8, immutable)
* `tuple`
* `ptr<T>`
* collections: {`list<T>`, `set<T>`, `map<T>`, `deque<T>`, `heap<T>`};

Value types:

* Are immutable by default
* Are copied efficiently
* Have no hidden allocation

---

## 5.3 Casting, Promotion, and Type Aliases

### 5.3.1 Casting Rules

Bestie enforces **explicit casting** except where the target type is already known.

Example:

```bestie
val x: int = 5 / 2        // integer division, result is int
```

However, implicit inference without an explicit target type is **illegal**:

```bestie
val x = 5 / 2             // ❌ compiler error
```

The programmer must make intent explicit:

```bestie
val x = (5 / 2) as int
```

Casting is always visible in source code and never implicit.

---

### 5.3.2 Type Promotion Is Explicit

Bestie does **not** allow implicit numeric promotion or narrowing.

Example:

```bestie
val x: int = 5
val y: byte = x           // ❌ compiler error
```

Correct form:

```bestie
val y: byte = x as byte
```

This rule applies **both directions**:

* Widening
* Narrowing

All promotions require an explicit `as` cast.

---

### 5.3.3 Type Aliases Using `as`

Bestie allows **type aliases** using the `as` keyword.

Aliases introduce **no new types** and incur **no runtime cost**.

Example:

```bestie
type UserId as int
type Bytes as list<byte>
```

Rules:

* Aliases are compile-time only
* Aliases improve readability and intent
* Aliases do not affect layout or ABI
* Casting rules still apply to the underlying type

---

## 6. Generics

Generics in Bestie are **compile-time constructs**, not runtime abstractions.

Characteristics:

* Full specialization (monomorphization)
* No type erasure
* No variance keywords
* No `extends?` or `super?`
* No wildcard types

Despite this, generics remain expressive through:

* Protocol constraints
* Explicit type relationships
* Compile-time resolution and diagnostics

Every generic instantiation produces concrete, optimized code with predictable layout and performance.

---

## 7. Control Flow: `if` and `switch`

### 7.1 `if` as Statement and Expression

In Bestie, `if` is both a **statement** and an **expression**.

As an expression, `if` must produce a value:

```bestie
val x: int = if (cond) 4 else 0
```

If a value does not have a natural empty representation, `option<T>` must be used.

As a statement, `if` may omit `else`.
When used inside a function without `else`, the function becomes **partial**:

```bestie
fun f(): bool? = if (cond) return true
```

See **fp.md** for partial functions.

---

### 7.2 `switch` as Statement and Expression

`switch` is both a **statement** and an **expression**.

Properties:

* No fallthrough
* Exhaustiveness checked when used as an expression
* Compact syntax (similar to modern Java switch)

```bestie
val x = switch (v) {
  1 => 10
  2 => 20
  else => 0
}
```

---

## 8. Loops

Bestie supports:

* `for`
* `for in`
* `while`

Loops may include an `else` clause, similar to Python.

Loops may also be used as **expressions** when resolvable at compile time:

```bestie
val x: int = for (i = 0; i < 5; i++) i + 5
val xs: list<int> = for (i in 0..3) i * 2
```

This provides comprehension-style behavior without stream abstractions.

---

## 9. Syntax Rules

* Parentheses `()` are required after `if`, `switch`, and loop keywords
* Braces `{}` are required when the body is not on the same line

```bestie
if (cond) doSomething() else doOther()

if (cond)
  doSomething() // ❌ compiler error
```

Bestie does **not** require `{` to be on the same line.
Formatting is enforced by the `fmt` tool.

---

## 10. Operators

Bestie supports a balanced mix of symbolic and word-based operators.

### Logical and Bitwise

* `&&`, `||`, `!`, `and`, `or`, `not`
* `&`, `|`, `^`, `~`, `<<`, `>>`

### Introspection and Identity

* `is` — identity comparison
* `typeOf(x)` — compile-time type query
* `sizeOf(T)` — compile-time size query

---

## 11. Functions (Overview)

Functions are declared using `fun`.

Properties:

* Static dispatch by default
* Explicit dynamic dispatch via annotations
* No implicit heap allocation

➡ See **fp.md**

---

## 12. Object-Oriented Programming (Overview)

Bestie supports OOP with explicit control:

* Closed classes by default
* Explicit inheritance
* Protocol-based polymorphism

➡ See **oop.md**

---

## 13. Memory Model (Overview)

Bestie uses manual, deterministic memory management:

* `ptr<T>`
* `own<T>`
* `ref<T>`

➡ See **memory.md**

---

## 14. Concurrency (Overview)

Concurrency is:

* Explicit
* Compile-time validated
* Free of hidden sharing

➡ See **concurrency.md**

---

## 15. Collections (Overview)

Core collections are deterministic, generic, and ownership-aware.

➡ See **collections.md**

---

## 16. Annotations

Annotations are compile-time only and introduce no runtime cost.

Examples:

* `@inline`
* `@virtual`
* `@override`
* `@immutable`

➡ See **annotation.md**

---

## 17. Error Handling (Overview)

Bestie avoids nulls and implicit exceptions.

Mechanisms:

* Complete returns
* Partial returns (`?`)
* `option<T>`
* Explicit error values

➡ See **errors.md**

---

## 18. Stability & Versioning

* Core language is sealed
* Backward compatibility is mandatory
* Experimental features require compiler flags
* APIs evolve without breaking core guarantees

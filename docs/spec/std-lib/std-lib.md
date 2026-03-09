# Standard Library (std-lib)

This document defines the **Bestie Standard Library (std-lib)**.

The standard library provides **portable, deterministic, and non–system-dependent functionality** built on top of the Bestie **core language**.

std-lib is intentionally **small**, **stable**, and **non-opinionated**.

---

## 1. Purpose of std-lib

std-lib exists to:

* Provide **canonical algorithms**
* Offer **pure utility abstractions**
* Avoid ecosystem fragmentation
* Remain fully portable across platforms
* Stay independent of OS, I/O, and runtime concerns

std-lib is **not** a framework and **not** a runtime.

---

## 2. What std-lib Is *Not*

std-lib explicitly does **not** include:

* Data structures (they are core language constructs)
* Error handling primitives (core language)
* Concurrency primitives (core language)
* I/O, filesystem, or networking (std-api)
* Dependency injection
* Logging frameworks
* Serialization frameworks
* Reflection or metaprogramming
* Background services or schedulers

---

## 3. Design Principles

std-lib strictly follows these rules:

1. **Pure by default**
2. **No hidden allocation**
3. **No global mutable state**
4. **Deterministic execution**
5. **Explicit inputs and outputs**
6. **No side effects unless clearly stated**
7. **No OS assumptions**

If behavior is not visible in the API, it does not exist.

---

## 4. Relationship to Core Language

### 4.1 Core Language Responsibilities

The core language provides:

* Primitive and composite types
* Data structures (`list`, `map`, `set`, etc.)
* Error handling model (`Result`, pattern matching)
* Concurrency primitives (`threadOs`, channels, mutexes)
* Memory and ownership model
* String type and templates

std-lib **builds on these**, but does not redefine them.

---

### 4.2 std-lib Responsibilities

std-lib provides:

* Algorithms
* Mathematical utilities
* Text processing helpers
* Deterministic helpers
* Common patterns expressed as pure code

---

## 5. Namespacing Rules

std-lib uses **stable, explicit namespaces**:

```text
std.algorithms
std.math
std.text
std.util
std.compare
std.pattern
```

Rules:

* No wildcard imports
* No implicit re-exports
* No aliasing of core symbols
* Namespaces are version-stable

---

## 6. Algorithms (`std.algorithms`)

Algorithms are:

* Stateless
* Free functions
* Generic
* Deterministic

Examples:

* `sort`
* `stableSort`
* `binarySearch`
* `min`, `max`
* `clamp`
* `partition`
* `fold`
* `zip`

Rules:

* No allocation unless explicitly documented
* Mutating and non-mutating variants are separate
* No implicit parallelism

Example:

```bestie
std.algorithms.sort(list)
```

---

## 7. Mathematics (`std.math`)

Provides:

* Mathematical constants
* Deterministic numeric algorithms
* Portable math utilities

Examples:

```bestie
math.PI
math.sqrt(x)
math.pow(a, b)
```

Rules:

* No platform-specific behavior
* No floating-point traps
* No hidden precision changes

---

## 8. Text Utilities (`std.text`)

Provides helpers for working with `str`:

* Trimming
* Splitting
* Searching
* Formatting helpers
* Parsing primitives

Rules:

* `str` is UTF-8
* No implicit encoding conversion
* Explicit normalization functions only

Example:

```bestie
std.text.trim(s)
std.text.split(s, ",")
```

---

## 9. Comparison Utilities (`std.compare`)

Provides:

* Total and partial ordering helpers
* Equality utilities
* Comparator builders

Used by:

* Sorting algorithms
* Ordered data structures
* User-defined ordering

---

## 10. Utility Helpers (`std.util`)

Provides:

* Range helpers
* Optional helpers (if applicable)
* Assertions (compile-time and runtime)
* Deterministic random generators (explicit seed)

Rules:

* No global RNG
* No hidden entropy sources

---

## 11. Patterns (`std.pattern`)

Provides **pure, language-level representations** of common patterns:

Examples:

* Builder helpers
* Visitor helpers
* Strategy helpers

Rules:

* No inheritance hierarchies
* No frameworks
* Patterns remain opt-in and explicit

---

## 12. Versioning

std-lib is versioned independently:

```text
bestie <lang-version>.<core-version>.<std-lib-version>.<std-api-version>
```

Rules:

* Breaking changes increment std-lib major version
* Patch versions never change behavior
* Experimental modules are explicitly marked

---

## 13. Stability Guarantees

Within a major version:

* APIs are stable
* Semantics do not change
* Performance regressions are treated as bugs

---

## 14. Summary

The Bestie standard library is:

* Small
* Pure
* Deterministic
* Portable
* Explicit

std-lib is a **mathematical and algorithmic foundation**, not a runtime.

Everything that touches the OS, I/O, networking, or concurrency coordination belongs elsewhere.

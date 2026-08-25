# Bestie Standard Patterns

This document defines **reusable design patterns** in Bestie as **protocols**, not frameworks.

Patterns in Bestie are:

* Explicit
* Compile-time driven
* Allocation-aware
* Optional (never forced)
* Expressed as contracts, not hierarchies

Bestie does not encode patterns into the language runtime.
Instead, it provides **precise protocol shapes** that allow patterns to emerge naturally and safely.

---

## Design Principles

1. **Patterns are protocols**
   Behavior is expressed via contracts, not base classes.

2. **No hidden control flow**
   Every pattern is resolved explicitly by the compiler.

3. **No runtime magic**
   No reflection, no RTTI, no dynamic proxies.

4. **Ownership-aware**
   Patterns must respect `own`, `ref`, and `ptr` semantics.

---

## 1. Iterator

The `Iterator<T>` protocol represents **explicit, pull-based iteration**.

```bestie
protocol Iterator<T> {
    fun next(): option<T>
}
```

### Semantics

* `next()` returns:

  * `Present(value)` when an element exists
  * `Not_Present` when iteration is complete
* Iterators are **stateful**
* No implicit allocation
* No hidden invalidation rules

### Rules

* Iterators may own or borrow their data source
* Thread safety depends on the underlying object
* Iterators are not restartable unless explicitly documented

---

## 2. Iterable

The `Iterable<T>` protocol represents **things that can produce iterators**.

```bestie
protocol Iterable<T> {
    fun iterator(): Iterator<T>
}
```

### Semantics

* `iterator()` creates a new iteration state
* No requirement for heap allocation
* Multiple iterators may coexist if the implementation allows it

### Relationship to Collections

* All std-lib collections `impl Iterable<T>`
* `Iterable<T>` does not imply mutability
* Iteration order is defined by the implementation

---

## 3. Factory

The `Factory<T>` protocol represents **object creation without exposing construction details**.

```bestie
protocol Factory<T> ext CreateFactory<T>, OfFactory<T, A> {
}

protocol CreateFactory<T> {
    fun create(): T
}

protocol OfFactory<T, A> {
    fun of(arg: A): T
}
```

### Semantics

* Encapsulates allocation and initialization
* May return:

  * Value types
  * Owned heap objects
  * Borrowed references (documented explicitly)

### Usage

Factories are preferred when:

* Construction logic is complex
* Allocation strategy may change
* Objects must be pooled or reused

Factories do **not** imply singleton behavior.

Additional `OfFactory` arities may be defined when a family of factories needs multiple explicit constructor shapes.

---

## 4. Builder

The `Builder<T>` protocol represents **step-by-step construction** of an object.

```bestie
protocol Builder<T> {
    fun build(): T
}
```

### Semantics

* Builders may be mutable
* `build()` consumes the builder by default
* Partial configuration is allowed

### Guidelines

* Builders should be short-lived
* Builders should not escape threads unless documented
* Builders are preferred over telescoping constructors

### Relationship to Factory

* A builder may internally use a factory
* A factory may return a builder

The patterns are complementary, not exclusive.

---

## 5. Proxy

The `Proxy<T>` protocol represents **controlled indirection**.

```bestie
protocol Proxy<T> {
    fun get(): ptr<T>
}
```

### Semantics

* `get()` returns a pointer to the underlying object
* Access may be:

  * Lazy
  * Validated
  * Synchronized
  * Remote

### Rules

* Proxies must not hide ownership transfer
* Proxies must document lifetime guarantees
* Proxies do not imply dynamic dispatch

### Common Uses

* Access control
* Lazy initialization
* Resource management
* Boundary enforcement

### Interaction with `copy` / `deepCopy`

A proxy is duplicated like any other object — by its **stored fields**, never by invoking `get()` (see `util.md` §7.7). Duplication therefore preserves a lazy proxy's unresolved state and follows the proxy's field qualifiers:

* If the proxy holds its target as an `own` field, `deepCopy` duplicates the target and `copy` is forbidden (it would duplicate ownership).
* If the proxy holds its target as `ref` or `ptr<T>` (the common case, since `get()` returns `ptr<T>`), both `copy` and `deepCopy` produce a proxy that **aliases the same target**. The target's lifetime remains the programmer's responsibility — it must outlive every proxy that points at it.

A proxy must not hide ownership transfer through duplication: copying a proxy never silently moves or frees the proxied object.

---

## 6. Singleton

Singleton is a **library pattern**, not a core language class kind.

```bestie
protocol Singleton<T> {
    fun instance(): ptr<T>
}
```

### Semantics

* Exactly one process-level instance for a given declaration
* Initialization is explicit and deterministic
* No implicit global object creation by language runtime

### Rules

* Implementations should use one-time init primitives (for example `Lazy<T>` or `Once<T>`)
* Cross-thread mutation remains explicit user responsibility
* No hidden synchronization beyond what the chosen primitive provides

### Why std-lib (not core)

* Keeps the core language small and predictable
* Avoids special singleton runtime semantics in class definitions
* Preserves explicit control over initialization, ownership, and synchronization

---

## Patterns Explicitly Excluded

The following patterns are **intentionally not provided** as core protocols:

* Observer
* Visitor
* Strategy
* Command

These are expressible via:

* Functions
* Lambdas
* Protocol composition

Providing them as built-ins would encourage unnecessary indirection.

---

## Summary

Bestie patterns are:

* Minimal
* Explicit
* Compile-time safe
* Ownership-aware

Patterns exist to **clarify intent**, not to add abstraction layers.

If a pattern cannot be expressed clearly as a protocol,
it does not belong in Bestie’s core or standard library.

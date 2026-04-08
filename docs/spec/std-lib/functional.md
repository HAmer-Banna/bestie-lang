# Functional Programming

This document defines the functional programming (FP) model in Bestie. FP in Bestie is intentionally **library-driven**, not runtime-driven, and avoids embedding behavior into data structures. Collections do **not** own methods such as `map` or `filter`; instead, functional operations are **free functions** imported from the standard functional library.

---

## Design Principles

Bestie’s functional model is guided by the following principles:

1. **No Runtime Dependency**
   Functional features are resolved at compile time and do not rely on a VM or runtime scheduler.

2. **Functions Over Methods**
   Operations such as `map`, `filter`, and `fold` are functions, not member methods. This preserves a clear separation between data and behavior.

3. **Explicit Imports**
   Functional capabilities are opt-in via imports. There is no implicit functional behavior on collections.

4. **Immutability by Default**
   Functional operations never mutate inputs. New values are always produced.

5. **Zero Magic**
   No implicit lifting, no hidden iterators, and no implicit parallelism.

---

## Functional Library

All functional operations are defined in:

```bestie
import bestie.lib.functional
```

Without this import, none of the functional symbols are visible.

---

## Core Functional Operations

### map

Transforms each element of a collection into a new value.

```bestie
val xs = list<int>.of(1, 2, 3);
val ys = map(xs, (x: int) => x * 2)
```

* `xs` is not modified
* `ys` is a new collection
* `map` is a free function

---

### filter

Selects elements that satisfy a predicate.

```bestie
val xs = list<int>.of(1, 2, 3, 4);
val ys = filter(xs, (x: int) => x % 2 == 0)
```

---

### fold

Reduces a collection into a single value.

```bestie
val sum = fold(xs, 0, (acc: int, x: int) => acc + x)
```

* The initial value is mandatory
* No implicit identity is assumed

---

## Lambdas

Lambdas are anonymous functions with explicit parameters.

```bestie
(x: int, y: int) => x + y
```

Rules:

* Lambdas are expressions
* Lambdas are strongly typed
* No implicit `this`

---

## Higher-Order Functions

Functions can accept and return other functions.

```bestie
fun twice(f: fn(int) -> int): fn(int) -> int {
    return (x: int) => f(f(x))
}
```

Higher-order behavior is resolved entirely at compile time.

---

## Partial Application

Bestie supports partial application explicitly.

```bestie
fun add(a: int, b: int): int = a + b
val add10 = add.bind(10)
```

* `bind` follows the core callable model
* Partial application never captures mutable state implicitly

---

## Currying

Currying is supported through function definitions, not syntax sugar.

```bestie
fun add(a: int): fn(int) -> int {
    return (b: int) => a + b
}
```

Currying is explicit and intentional.

---

## Composition

Function composition is provided as a library utility.

```bestie
val f: fn(int) -> int = (x: int) => x + 1
val g: fn(int) -> int = (x: int) => x * 2
val h = compose(f, g)
```

Execution order is explicit and left-to-right.

---

## Functional Collections

Functional operations work on any collection type that satisfies the **iterable protocol**.

* `list`
* `set`
* `deque`
* `map`

Collections themselves remain structurally simple and behavior-free.

---

## No Method Chaining

The following is **intentionally invalid**:

```bestie
xs.map((x: int) => x * 2)
```

This restriction:

* Prevents hidden allocations
* Keeps data structures minimal
* Avoids fluent-style overengineering

---

## Error Handling

Functional error handling is expressed via explicit types, not exceptions.

```bestie
val results = map(xs, (x: int) => safeDivide(10, x))
```

Propagation rules are defined by the involved types (`!`, `option`, `result`), not by the functional system.

---

## Summary

* Functional programming in Bestie is **explicit, minimal, and compile-time driven**
* Collections do not own behavior
* All functional power lives in `bestie.lib.functional`
* No runtime, no magic, no implicit parallelism

This model keeps Bestie predictable, portable, and suitable for environments ranging from bare metal to large-scale systems.

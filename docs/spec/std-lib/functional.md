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

All three take their source as a generic parameter constrained by `Iterable<T>`, never as a concrete collection type. Variations are invariant (`collections.md` §3.3), so a `List<T>` parameter would accept array-backed lists only; the constraint accepts every variation, plus core `array<T>`, `slice<T>`, and `range<T>`.

They take their callable the same way — as a generic parameter constrained by a function shape (`core/fp.md` §6.5):

```bestie
fun <C impl Iterable<T>, T, R, F impl fn(T) -> R> map(xs: C, f: F): List<R>
fun <C impl Iterable<T>, T, P impl fn(T) -> bool> filter(xs: C, p: P): List<T>
fun <C impl Iterable<T>, T, R, F impl fn(R, T) -> R> fold(xs: C, initial: R, op: F): R
```

`map` and `filter` return an array-backed `List<R>` — a new, owned collection the caller must discharge. They are monomorphized per collection type **and per callable**, so the lambda is a direct call inlined into the loop: `map(xs, x => x * 2)` compiles to the loop a C programmer would write. Lambda parameter types come from the signature (`core/fp.md` §7.2).

### map

Transforms each element of a collection into a new value.

```bestie
val xs = List<int>.new(1, 2, 3);
val ys = map(xs, x => x * 2)
```

* `xs` is not modified
* `ys` is a new collection
* `map` is a free function

---

### filter

Selects elements that satisfy a predicate.

```bestie
val xs = List<int>.new(1, 2, 3, 4);
val ys = filter(xs, x => x % 2 == 0)
```

---

### fold

Reduces a collection into a single value.

```bestie
val sum = fold(xs, 0, (acc, x) => acc + x)
```

* The initial value is mandatory
* No implicit identity is assumed

---

## Lambdas

Lambdas are anonymous functions. Their parameter types are written, or taken from the function shape the lambda is passed to (`core/fp.md` §7.2).

```bestie
(x: int, y: int) => x + y
map(xs, x => x + 1)
```

Rules:

* Lambdas are expressions
* Lambdas are strongly typed — each has its own type (`core/fp.md` §6.2)
* No implicit `this`, and no implicit captures — a capture is written in a `[...]` list

---

## Higher-Order Functions

Functions accept other functions as generic callables (`core/fp.md` §6.5), or as a `fn[]` view when one compiled copy is wanted (`core/fp.md` §6.4):

```bestie
fun <F impl fn(int) -> int> twice(f: F, x: int): int = f(f(x))

twice(x => x + 3, 10)        // 16 — two direct calls, inlined
```

Returning a function that carries data needs a type for it. A lambda's own type cannot be written in a signature, and a `fn[]` cannot outlive what it views, so a function that builds a callable returns either a named type that implements a function shape (`core/fp.md` §6.7) or a `Closure`.

---

## Partial Application

Partial application is written as a lambda with an explicit capture list. There is no `bind` (`core/fp.md` §13).

```bestie
fun add(a: int, b: int): int = a + b

val ten = 10
val add10 = [ten](b: int) => add(ten, b)      // no expected shape here, so b is typed
val ys = map(xs, [ten](b) => add(ten, b))      // passed to map: b's type comes from map
```

* The capture list states what is carried, at the site where it happens
* Partial application never captures mutable state implicitly

---

## Currying

There is no currying syntax. A function that returns a configured function returns a named callable:

```bestie
value class Adder impl fn(int) -> int {
    val n: int
    fun call(x: int): int = x + this.n
}

fun adder(n: int): Adder = Adder.new(n: n)

val add5 = adder(5)
add5(10)                     // 15 — add5.call(10), a direct call
```

Currying is explicit and intentional, and costs exactly the bytes of what it carries.

---

## Composition

```bestie
fun <A, B, C, F impl fn(B) -> C, G impl fn(A) -> B> compose(f: F, g: G): Composed<A, B, C, F, G>
```

`compose(f, g)` returns a `Composed` value holding `f` and `g` inline; calling it calls `g`, then `f`, both directly (`core/fp.md` §12).

```bestie
val inc = (x: int) => x + 1
val dbl = (x: int) => x * 2
val h = compose(inc, dbl)
h(5)                         // 11 — inc(dbl(5))
```

The argument order is mathematical: `compose(f, g)(x)` is `f(g(x))`.

---

## `Closure` — A Heap-Owned Callable

Core's callables are free: a function pointer, a generic callable, a view (`core/fp.md` §6). None of them can hold callables **of different types with captures** in one slot — a list of event handlers, a task queue, a callback field whose value is chosen at run time. That needs the captures on the heap, so it is a library type with a named allocation:

```bestie
import bestie.lib.functional.Closure

class Closure<S> impl S {                         // S is a function type: Closure<fn(Event) -> void>
    fun <F impl S> new(own f: F): own Closure<S>  // allocates; moves the callable into the block
    fun freeDeep(): void                          // discharges the captures, then frees the block
}
```

```bestie
class EventBus {
    val own handlers: List<Closure<fn(Event) -> void>>

    fun subscribe(own h: Closure<fn(Event) -> void>) { this.handlers.add(move h) }
    fun publish(e: Event) {
        for (h in this.handlers) { h(e) }        // one indirect call per handler
    }
}

val own log = try File.open(path)
bus.subscribe(Closure<fn(Event) -> void>.new([move log](e) => log.writeLine(e.name)))
bus.subscribe(Closure<fn(Event) -> void>.new(e => metrics.count(e.kind)))
```

Representation and rules:

* Two words in the holder: the code address of a trampoline instantiated for `F`, and the address of a heap block holding `F`'s captures and the function that discharges them. A non-capturing `F` allocates nothing — the block address is unused.
* `Closure<S>` implements `S`, so it is called like a function (`core/fp.md` §6.7) and converts to a `fn[]` view.
* It is an `own` value: moved, not copied, and discharged with `freeDeep()`, which first discharges the captures (`core/fp.md` §7.7). Forgetting to is the ordinary leak error.
* It allocates through the current allocator, like any `.new()`.
* It is built entirely from core pieces — a generic callable, a function pointer, and `ptr<T>` — so code that never imports it pays nothing (`vision.md` Rule 4).

Choose it only when the slot's callable type varies at run time. A field of one known callable type is a generic `F` (`core/fp.md` §6.5) and needs no allocation.

---

## Functional Collections

Functional operations work on anything satisfying `Iterable<T>` (`utils.md` §2):

* every `List` / `Set` / `Map` / `Deque` / `PriorityQueue` variation, including `.immutable`
* core `array<T>`, `slice<T>`, and `range<T>`

That reach is the reason the constraint is written as `impl Iterable<T>` rather than as a concrete parameter type — a `List<int>` parameter could take none of the others.

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

Propagation rules are defined by the involved types (`T ?`, `T ! E`), not by the functional system.

---

## Summary

* Functional programming in Bestie is **explicit, minimal, and compile-time driven**
* Collections do not own behavior
* All functional power lives in `bestie.lib.functional`
* Higher-order functions take generic callables, so lambdas inline; `Closure` is the one heap-allocating callable, and its `.new()` says so
* No runtime, no magic, no implicit parallelism

This model keeps Bestie predictable, portable, and suitable for environments ranging from bare metal to large-scale systems.

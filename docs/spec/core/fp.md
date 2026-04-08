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

Return type modifiers may be combined:

| Signature                        | Meaning                                      |
| -------------------------------- | -------------------------------------------- |
| `fun f(): T`                     | Always returns `T`                           |
| `fun f(): T ?`                   | May not return (partial)                     |
| `fun f(): T ! E`                 | Returns `T` or error from set `E`            |
| `fun f(): T ! `                  | Returns `T` or inferred error set            |
| `fun f(): (T, U)`                | Returns multiple values as tuple             |

These modifiers are **mutually exclusive per return slot**. A function may not return both `?` and `!` on the same value.

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

`T ?` is syntactic sugar for `option<T>` as a return type. A partial function either returns a value (wrapped in `option.Present`) or returns nothing (`option.Not_Present`). The `?` modifier is a declaration-site shorthand — the caller always receives an `option<T>`.

```bestie
fun getUser(id: int): User ?   // equivalent to: fun getUser(id: int): option<User>
```

Inside the function body:
- `return value` → compiler wraps as `option.Present(value)`
- bare `return` → compiler emits `option.Not_Present`

```bestie
fun getUser(id: int): User ? {
    if (exists(id)) {
        return repository.find(id)   // → option.Present(user)
    }
    return                           // → option.Not_Present
}
```

Rules:

* Caller always receives `option<T>` — no hidden null, no truthiness check
* Compiler enforces exhaustiveness — the absent case must be handled
* No implicit exceptions, no sentinel values

---

#### 3.3 Calling Partial Functions

The result of a partial function is an `option<T>`. It must be explicitly handled. There is no truthiness check — Bestie has no null and no implicit presence test.

**Pattern matching — full control:**

```bestie
switch (getUser(id)) {
    case option.Present(val user) => sendEmail(user)
    case option.Not_Present       => println("user not found")
}
```

**`if`-let — bind and enter block only when present:**

```bestie
if (val user = getUser(id)) {
    sendEmail(user)   // user is User here, not option<User>
}
```

**`else` unwrap — provide a fallback or divert:**

```bestie
val user = getUser(id) else { return }           // absent → early return
val user = getUser(id) else { User.anonymous() } // absent → fallback value
```

Compiler enforces:

* `option<T>` cannot be used directly where `T` is expected — unwrapping is required
* All branches of `switch` on `option<T>` must be covered
* `if`-let binds the inner `T` value — the outer `option<T>` is not accessible inside the block

### 3.4 Lambdas and Partiality

Lambdas may also be complete or partial.

Complete lambda:

```bestie
val inc = (x: int) => x + 1;
```

Partial lambda:

```bestie
val findUser = (x: int): User ? => if (x > 0) return repo.get(x);
```

---

## 4. No-Null Guarantee

Bestie has no `null`, no `nil`, no `NULL`, and no `nullptr`. These concepts do not exist in the language at any level.

This promise holds across all layers of the system:

---

### 4.1 Pure Bestie Code

There is no null literal. There is no nullable type. There is no way to write null in a Bestie source file. The compiler has no concept of a "null value" — only `option.Not_Present` for the absent case.

| In other languages | In Bestie |
| ------------------ | --------- |
| `null`, `nil`, `nullptr` | Does not exist |
| `T?` (nullable reference) | `option<T>` (explicit, typed) |
| Null pointer dereference | Cannot occur through safe code |
| Sentinel `-1` or `0` for "no value" | `option<T>` |
| Returning `null` | `return` (bare) in a `T ?` function |

---

### 4.2 `ptr<T>` — Raw Pointer

`ptr<T>` is Bestie's escape hatch for unsafe memory operations. It can physically hold any bit pattern, including the zero address. However:

* The zero address in a `ptr<T>` is not called "null" — it is an **invalid address**
* Safe Bestie code never produces a zero-address `ptr<T>`
* The compiler's niche optimization uses the zero address bit pattern internally — this is invisible to the programmer
* Any `ptr<T>` that might hold the zero address comes from `foreign` code or `@trusted` unsafe operations — both are explicit in source

---

### 4.3 Third-Party Bestie Libraries

A library written in Bestie is structurally incapable of introducing null. It can only express absence via `option<T>` or `T ?`. This is enforced by the type system — not a convention, not a guideline.

---

### 4.4 FFI / Foreign C Libraries

C functions that return nullable pointers are mapped to `option<ptr<T>>` at the FFI boundary. The FFI layer converts C `NULL` (zero address) → `option.Not_Present` automatically. The `null` keyword is not used in the foreign function declaration — see `std-api/foreign.md` §7.

---

### 4.5 The Structural Guarantee

No code path in a Bestie program — core, stdlib, third-party, or FFI wrapper — can return a value typed as something the caller must null-check. Every absent case is an explicit `option.Not_Present` handled at the call site through pattern matching or an `else` unwrap. The compiler enforces this exhaustively.

---

## 5. Multiple Return Values and Tuples


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

## 6. Function Types

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
* Callable values lower to a light callable representation
* Non-capturing functions lower to direct code references
* Capturing callables lower to fixed-size compiler-known callable objects
* No vtables or heap-managed closure objects

---

## 7. Lambdas

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

### 6.3 Explicit Immutable Capture

```bestie
val factor: int = 3
val f = [factor](x: int) => x * factor
```

Rules:

* Captured values are copied at the point of lambda creation
* Captures are immutable — cannot be reassigned inside the lambda
* `own` values cannot be captured
* Capture layout is compile-time known
* No heap allocation introduced

---

### 6.4 Explicit Mutable Capture

Mutable captures are allowed via `[var x]`. The lambda receives a mutable copy that persists between calls.

```bestie
var count = 0
var counter = [var count](x: int): int => {
    count += x
    return count
}

counter(5)   // returns 5  — internal count = 5
counter(3)   // returns 8  — internal count = 8
```

The original `count` is **unaffected** — the lambda owns its own mutable copy.

Rules:

* `[var x]` captures a mutable copy — the original binding is unchanged
* Mutations to `var` captures persist between invocations
* Cannot be shared across threads — mutable state forbids it
* May be passed, stored, and returned as a light callable value
* `own` values cannot be captured as `var`
* No heap allocation — captured state is inline in the callable value
* Capture layout is compile-time known

---

### 6.5 Lambda Allocation Model

* Lambdas do not heap-allocate — compile-time lowered to inline code or light callable values
* Immutable captures — part of a zero-size or fixed-size compile-time-known callable context
* Mutable captures — inline in the callable value, no heap
* Heap allocation for lambdas is **not part of the core language**

---

## 8. Higher-Order Functions

```bestie
fun apply(f: fn(int) -> int, x: int): int {
    return f(x)
}
```

Rules:

* Function arguments are explicit
* No runtime boxing
* Calls are direct or inlined when the callee is known at compile time
* Indirect call occurs only when a callable value is actually passed around
* Fully compatible with ownership and concurrency rules

---

## 9. Variable Arguments (Varargs)

Variadic parameters are declared with `...` after the element type.
`val` and `var` carry their normal semantics — `val` is the default and preferred form.

```bestie
fun sum(val xs: int...): int {
    var total = 0
    for x in xs {
        total += x
    }
    return total
}
```

Mutable varargs allow modification of elements inside the function body:

```bestie
fun normalize(var xs: float64...): void {
    for i in 0..xs.size {
        xs[i] = xs[i] / 100.0
    }
}
```

Usage:

```bestie
sum(1, 2, 3, 4)
normalize(10.0, 20.0, 50.0)
```

Rules:

* `...` after the element type marks the parameter as variadic
* `val xs: T...` — immutable variadic, elements cannot be reassigned (default)
* `var xs: T...` — mutable variadic, elements may be modified inside the function
* `const` is **not allowed** for varargs — variadic arguments are runtime values
* Element type must be explicitly specified
* Only the last parameter may be variadic
* Argument sequence is stack-allocated when possible
* No implicit heap allocation is introduced

---

## 10. Method References

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

### 9.2 Bound Method References

A bound method reference binds an instance to a method, producing a zero-argument light callable.
The behavior depends on the type of the bound object.

---

**Value types** (`data class`, `value class`, `enum`, primitives) — copy:

```bestie
val p = Point(x = 1, y = 2)
val f: fn() -> int = p::getX    // ✅ p is copied into f
```

The value is copied into the callable's inline context at the point of binding. No ownership concerns.

---

**`own` values — explicit `move` required:**

```bestie
val own u = User.new()
val f: fn() -> str = move u::getName   // ✅ u is moved into f
f()                                     // calls getName on the moved user

use(u)   // ❌ compile error: u has been moved
```

`move` transfers ownership into the callable's inline context. The source binding becomes invalid immediately — consistent with all other ownership transfer in Bestie.

Without `move`, binding an `own` value is a **compile-time error**:

```bestie
val own u = User.new()
val f: fn() -> str = u::getName   // ❌ compile error: u is own, use 'move u::getName'
```

**`ref` values — forbidden:**

`ref` cannot escape scope or be stored, so binding a `ref` to a method reference is always a compile-time error:

```bestie
fun bad(user: ref User) {
    val f = user::getName   // ❌ compile error: ref cannot be captured
}
```

Use an unbound reference and pass the value explicitly instead:

```bestie
fun good(user: ref User) {
    val f: fn(ref User) -> str = User::getName
    f(user)   // ✅
}
```

---

Rules summary:

* Value types — copied at binding site, no ownership tracking needed
* `own` values — `move` required, source becomes invalid
* `ref` values — forbidden, use unbound references instead
* No allocation introduced in any case
* Lowered at compile time

---

## 11. Extension Functions

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

* Do not use `impl` to satisfy protocols
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

## 12. Function Composition

```bestie
fun compose<A, B, C>(
    f: fn(B) -> C,
    g: fn(A) -> B
): fn(A) -> C {
    return (x: A) => f(g(x))
}
```

No implicit currying or composition exists.

---

## 13. Currying and Partial Application

### 12.1 No Implicit Currying

Automatic currying is not supported.
Capturing-based currying is illegal.

### 12.2 Explicit Partial Application

`bind` performs partial application — it fixes one or more arguments of a function, producing a new function with fewer parameters.

Bestie resolves `bind` **at compile time whenever possible**. When the bound value is only known at runtime, it is captured by copy — same semantics as explicit lambda capture, stored in a fixed-size callable context, no heap.

**Compile-time constant — fully resolved at compile time:**

```bestie
val add5 = add.bind(5)       // 5 is a constant — fully specialized, zero runtime presence
add5(3)                       // compiles to add(5, 3) directly
```

**Runtime value — captured by copy in the callable context:**

```bestie
val n = getUserInput()
val addN = add.bind(n)       // n is runtime — n is copied into addN
addN(3)                       // compiles to add(n_copy, 3)
```

Equivalent to writing the explicit capture form:

```bestie
val addN = [n](b: int) => add(n, b)
```

**Binding multiple arguments:**

```bestie
fun clamp(min: int, max: int, val: int): int
val clamp0to100 = clamp.bind(0, 100)   // fixes min and max
clamp0to100(42)                         // compiles to clamp(0, 100, 42)
```

Rules:

* Compile-time constant arguments → fully specialized at compile time
* Runtime arguments → captured by copy, immutable, inline in the callable value
* No heap allocation in either case
* Captured values follow the same rules as explicit lambda captures — value types only, no `own`, no `ref`
* `bind` arguments are bound left to right

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

Ensures:

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

## 17. Immutability in FP

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

## 18. What Bestie Deliberately Avoids in FP

* Implicit currying
* Lazy evaluation by default
* Runtime monads
* Hidden effect systems
* Reflection-based dispatch
* Closure-heavy abstractions

---

## 19. Summary

Functional programming in Bestie is:

* Explicit
* Compile-time resolvable
* Allocation-aware
* Ownership-safe
* Concurrency-safe
* Zero-cost by design

FP in Bestie exists to **compose behavior clearly**, not to obscure execution.

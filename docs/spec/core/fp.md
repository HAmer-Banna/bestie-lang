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

* Hidden / environment closures (implicit upvalues over outer frames)
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
* Capture only when written in source (`[x]` / `[var x]` on lambdas — never on local `fun`)

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
* Local functions (nested inside a function, method, or block — see §2.4)

Return type modifiers may be combined:

| Signature                        | Meaning                                      |
| -------------------------------- | -------------------------------------------- |
| `fun f(): T`                     | Always returns `T`                           |
| `fun f(): T ?`                   | May be absent (`T ?` — core syntax)          |
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

### 2.4 Local Functions

Local functions are **named helpers nested inside a function, method, or block**. They exist for locality and encapsulation — not for closing over outer state.

They parallel inner classes (`core/oop.md` §8): **lexically nested, not implicitly bound**.

```bestie
fun process(items: list<int>): int {
    fun clamp(x: int): int {
        if (x < 0) { return 0 }
        if (x > 100) { return 100 }
        return x
    }

    var total = 0
    for item in items {
        total += clamp(item)
    }
    return total
}
```

#### Placement and visibility

* A local `fun` may appear in any block scope (function body, method body, nested block)
* Within a block, all local function names declared in that block are visible to each other for the whole block (supports mutual recursion without forward declarations)
* A local function is not visible outside its enclosing block
* Local functions are **never exportable** (see `modules-and-packaging.md` §7.2)
* Nested local functions are allowed (a local `fun` may declare further local `fun`s)
* A local function may not shadow another local function in the same block; shadowing an outer local function across nested blocks follows normal scope shadowing (`base.md` §4.8)

#### Non-closure rule (same default as lambdas)

By default a local function captures **nothing**. It may access only:

* Its own parameters
* Compile-time constants (`const`)
* Top-level / module-visible pure functions and types

Accessing an enclosing local binding without passing it as a parameter is a **compile-time error**:

```bestie
fun outer(n: int): int {
    fun bad(): int {
        return n   // ❌ compile error: local function cannot see outer locals
    }
    return bad()
}

fun outerOk(n: int): int {
    fun scale(x: int, factor: int): int = x * factor
    return scale(2, n)   // ✅ pass explicitly
}
```

Local functions **do not support capture lists**. Capturing is exclusively the domain of lambdas (`[x]` / `[var x]`, §7) and related callable construction (`bind`, bound method references). If a named capturing callable is needed:

```bestie
fun outer(factor: int): int {
    val scale = [factor](x: int) => x * factor   // named lambda — explicit capture
    return scale(10)
}
```

#### Nested methods — not a separate construct

There is no distinct “nested method” form that implicitly binds outer `this`.

* Inside a method body, a nested `fun` is a **local function**, not a method
* It does **not** receive or capture `this` implicitly
* To use the receiver, pass `this` (or a field) as an ordinary parameter

```bestie
class Counter {
    var value: int = 0

    fun bumpTwice(): int {
        fun step(c: Counter): void {
            c.value += 1
        }
        step(this)
        step(this)
        return this.value
    }
}
```

#### Signature and semantics

Local functions use the same declaration surface as other `fun`s:

* Explicit parameter and return types
* Expression-body form (`= expr`)
* Complete / partial (`T ?`) / error (`T ! E`) returns
* Default parameters and keyword arguments
* Recursion by name (including mutual recursion in the same scope)

#### Lowering model

* Lowered to a **mangled static function** in the enclosing compilation unit
* Static dispatch; direct `call` (or inline) when the callee is known
* Taking the address of a local function, storing it, or passing it as `fn(T) -> R` uses the same thin/fat callable model as any named function (§6) — non-capturing, so thin / null-context fat
* No trampolines, no heap frame, no runtime code generation
* No hidden allocation

#### Local function vs lambda

| | Local `fun` | Lambda |
| --- | --- | --- |
| Name | Named, recursive by name | Anonymous (bind to `val`/`var` for a name) |
| Capture | Never — pass parameters | None by default; explicit `[x]` / `[var x]` |
| Role | Scoped static helper | Inline / first-class behavior value |
| Lowering | Mangled static function | Thin or fat callable (§6) |

**Design intent:** use local functions for **naming and scope**; use lambdas when you need a **callable value** or **explicit capture**. Nesting must not become a back door into Python-style environment closures.

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

`T ?` is core syntax (see `types.md` §8.3). On a function return it is the natural spelling: a partial function either returns a value or returns nothing.

```bestie
fun getUser(id: int): User ?
```

Inside the function body:
- `return value` → the result is present
- bare `return` → the result is absent

```bestie
fun getUser(id: int): User ? {
    if (exists(id)) {
        return repository.find(id)
    }
    return
}
```

Rules:

* Caller always receives `User ?` — no hidden null, no truthiness check
* Compiler enforces exhaustiveness — the absent case must be handled
* No implicit exceptions, no sentinel values
* Optional **parameters and fields** use the same syntax: `fun connect(host: str, port: int ?)`, `val token: str ?`. `?` is not return-only.

The named type `option<T>` and constructors `option.Present` / `option.Not_Present` live in `bestie.lib.utilities`. They are the same representation as `T ?`, not a second system. Import them to match by name; `if-let` below needs no import.

---

### 3.3 Calling Partial Functions

The result of a partial function is `T ?`. It must be explicitly handled. There is no truthiness check — Bestie has no null and no implicit presence test.

**`if`-let — core, no import.** Bind and enter the block only when present:

```bestie
if (val user = getUser(id)) {
    sendEmail(user)   // user is User here, not User ?
}
```

**`else` unwrap — core.** Provide a fallback or divert:

```bestie
val user = getUser(id) else { return }           // absent → early return
val user = getUser(id) else { User.anonymous() } // absent → fallback value
```

**Named matching — std-lib.** Import `bestie.lib.utilities` to match constructors:

```bestie
import bestie.lib.utilities.option

switch (getUser(id)) {
    case option.Present(val user) => sendEmail(user)
    case option.Not_Present       => println("user not found")
}
```

Compiler enforces:

* `T ?` cannot be used directly where `T` is expected — unwrapping is required
* All branches of `switch` on `option<T>` must be covered
* `if`-let binds the inner `T` value — the outer `T ?` is not accessible inside the block

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

There is no null literal. There is no nullable type. There is no way to write null in a Bestie source file. The compiler has no concept of a "null value" — only absence of `T ?`.

| In other languages | In Bestie |
| ------------------ | --------- |
| `null`, `nil`, `nullptr` | Does not exist |
| `T?` (nullable reference) | `T ?` (core syntax; named `option<T>` in std-lib) |
| Null pointer dereference | Cannot occur through safe code |
| Sentinel `-1` or `0` for "no value" | `T ?` |
| Returning `null` | `return` (bare) in a `T ?` function |

---

### 4.2 `ptr<T>` — Raw Pointer

`ptr<T>` is Bestie's raw, low-level pointer for direct memory operations. It can physically hold any bit pattern, including the zero address. However:

* The zero address in a `ptr<T>` is not called "null" — it is an **invalid address**
* `own`/`ref` code never produces a zero-address `ptr<T>`
* The compiler's niche optimization uses the zero address bit pattern internally — this is invisible to the programmer
* Any `ptr<T>` that might hold the zero address comes from `foreign` code or `@trusted` operations — both are explicit in source

---

### 4.3 Third-Party Bestie Libraries

A library written in Bestie is structurally incapable of introducing null. It can only express absence via `T ?` (or the named std-lib form `option<T>`). This is enforced by the type system — not a convention, not a guideline.

---

### 4.4 FFI / Foreign C Libraries

C functions that return nullable pointers are mapped to `ptr<T> ?` at the FFI boundary. The FFI layer converts C `NULL` (zero address) to absent automatically. The `null` keyword is not used in the foreign function declaration — see `std-api/foreign.md` §7.

---

### 4.5 The Structural Guarantee

No code path in a Bestie program — core, std-lib, third-party, or FFI wrapper — can return a value typed as something the caller must null-check. Every absent case is `T ?`, handled at the call site through `if-let`, `else` unwrap, or (with std-lib names) pattern matching. The compiler enforces this exhaustively.

---

## 5. Multiple Return Values and Tuples


Bestie allows functions to return **multiple values** using **tuples**.

### 5.1 Tuple Return Types

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

### 5.2 Tuple Return Shortcut

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

### 5.3 Tuple Destructuring (Capture Shortcut)

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

### 5.4 Ignoring Values with `_`

```bestie
val quotient, _ = divMod(10, 3)
```

Rules:

* `_` introduces no binding
* Ignored values are not accessible
* Helps document intent and avoid unused-variable diagnostics

---

## 6. Function Types

Function types are **explicit and structural**. They are written with a single arrow `->` from the parameter list to the return type:

```bestie
(int) -> int
```

Usage:

```bestie
val f: (int) -> int = square
```

### 6.1 The Optional `fn` Prefix

The keyword `fn` may **optionally** prefix a function type. Both forms are exactly equivalent — `fn` adds no semantics, only emphasis:

```bestie
val f: (int) -> int    = square    // bare form
val g: fn(int) -> int  = square    // fn-prefixed form — identical type
```

There is no ambiguity to resolve here, so the prefix is never *required*:

* A **function type** uses the single arrow `->` (`(int) -> int`).
* A **lambda** (a value/expression) uses the fat arrow `=>` (`(x: int) => x * 2`).
* A **tuple type** has no arrow at all (`(int, str)`).

The arrows alone fully disambiguate the three. The `fn` prefix exists purely so a reader (or a `grep`) can spot a function type at a glance.

**Style guideline:**

* Prefer the **bare** form `(T) -> R` in simple signatures — it is lighter and reads cleanly.
* Reach for the **`fn`-prefixed** form in dense or nested signatures, where a leading `fn` makes a higher-order parameter or a returned function easier to pick out:

```bestie
// nested / higher-order — fn aids readability
fun compose(f: fn(int) -> int, g: fn(int) -> int): fn(int) -> int =
    (x: int) => f(g(x))

// simple — bare form is enough
fun apply(f: (int) -> int, x: int): int = f(x)
```

Both spellings are accepted everywhere a function type is valid: parameters, return types, `val`/`var` bindings, and generic arguments. The examples throughout this document use the `fn` form for explicitness; they are equivalent to dropping the prefix.

### 6.2 Lowering Model

Every callable value has a concrete compile-time lowering. There are two representations:

---

**Thin callable** — a bare function pointer. One word.

Used when:

* The callable is a named function (top-level, method, extension, or local), or a non-capturing lambda
* The callee is statically known at the call site and inlined or directly called

```
[ code_ptr ]   (1 word)
```

At a direct call site where the callee is statically known, the compiler emits a direct `call` instruction. No representation is materialized at all.

---

**Fat callable** — a code pointer plus a context pointer. Two words.

Used when:

* The callable captures one or more values (explicit `[x]` or `[var x]` captures)
* A bound method reference carries an instance (copy or moved `own`)
* A `bind()` result carries runtime-bound arguments

```
[ code_ptr | context_ptr ]   (2 words)
```

`context_ptr` points to the **capture struct**, which is a fixed-size, stack-allocated record holding the captured values in capture-declaration order.

For bound method references with `own` move, the context struct holds the moved value inline. For value-type copies, the struct holds the copied value inline.

`context_ptr` is a typed `ptr<CaptureStruct>`. The `CaptureStruct` type is anonymous and compiler-generated — not accessible in user code.

---

**Unification at higher-order function call sites:**

When a callable is passed to or stored in a `fn(T) -> R` typed location, the fat two-word representation is always used. Non-capturing callables placed in this context carry a `context_ptr` of null (zero address). The callee-side calling convention checks `context_ptr != null` before dereferencing.

This means:

* Passing a non-capturing lambda to `fun apply(f: fn(int) -> int)` — `f` is fat with null context
* Passing a capturing lambda — `f` is fat with a live context pointer
* The call through `f` uses the indirect calling convention in both cases

When the callee type is fully known at the call site (monomorphic HOF or inlined call), the compiler may specialize and emit a direct call without the fat-pointer indirection.

---

**Capture struct lifetime:**

The capture struct is stack-allocated at the lambda or `bind()` creation site. It lives as long as the callable value is live. The callable may not outlive the scope of its capture struct. The compiler enforces this through the same scope-based liveness analysis used for `ref`.

---

**Calling convention summary:**

| Callable kind | Representation | Call instruction |
| ------------- | -------------- | ---------------- |
| Named function (incl. local), statically known | none (direct) | `call fn_ptr` |
| Non-capturing lambda, statically known | thin (1 word) or none | `call fn_ptr` |
| Non-capturing lambda, passed as `fn(T)->R` | fat (2 words, null context) | indirect call |
| Capturing lambda (explicit `[...]` only) | fat (2 words, live context) | indirect call |
| Bound method — value copy | fat (2 words, inline context) | indirect call |
| Bound method — `own` move | fat (2 words, inline context) | indirect call |
| `bind()` with runtime args | fat (2 words, live context) | indirect call |

---

## 7. Lambdas

Lambdas are anonymous functions with **explicit and restricted semantics**.

### 7.1 Lambda Syntax

```bestie
val f = (x: int): int => x * 2
```

Properties:

* Parameter types are explicit
* Return type inferred from body
* No implicit allocation
* Compile-time lowered

---

### 7.2 Non-Closure Rule (Core Guarantee)

**Bestie does not have environment closures.**

In Python, JavaScript, Kotlin, and similar languages, a nested function or lambda silently forms a *closure* over its lexical environment: free variables resolve to the outer frame (often by reference), lifetimes extend with the callable, and allocation / sharing costs are hidden.

Bestie rejects that model. By default, lambdas and local functions:

* Capture nothing
* Access only:

  * Their parameters
  * Compile-time constants (`const`)
  * Module-visible pure functions and types

Illegal without an explicit capture list:

```bestie
val y = 10
val f = (x: int) => x + y   // compile-time error — y is not captured
```

Guarantees:

* No hidden sharing
* No outer-frame / upvalue machinery
* No lifetime complexity beyond ordinary scope rules
* Concurrency safety
* Zero allocation by default

---

### 7.3 Closures in Bestie — Evaluation and Position

Bestie’s position is deliberate and locked to the platform pillars (`docs/platform.md`):

| Model | Status in Bestie |
| --- | --- |
| Implicit lexical closures (Python / JS / Kotlin default) | **Rejected** |
| Closures that share / mutate the outer binding by reference | **Rejected** |
| Closures that heap-allocate an environment object | **Rejected** in core |
| Escaping nested functions via trampolines / executable stacks | **Rejected** |
| Explicit capture-by-copy on lambdas (`[x]`, `[var x]`) | **Allowed** — the only capturing form |
| Local `fun` with free outer locals | **Rejected** — pass parameters (§2.4) |

What Bestie *does* provide is not an environment closure. It is an **explicit capture callable**:

* Every captured name appears in a `[...]` list at the creation site
* Capture is **by value (copy)** into a fixed-size, stack-allocated capture struct (§6)
* `[var x]` still copies — it creates **private mutable state inside the callable**, not a live alias to the outer `var`
* The callable may not outlive its capture struct (same scope liveness analysis as `ref`)
* Cost and layout are visible and compile-time known

```bestie
var count = 0
val counter = [var count](x: int): int => {
    count += x
    return count
}
count = 100
counter(1)   // returns 1 — outer `count` was never shared
```

**Why this stays pillar-safe**

* **Compilation speed** — no escape-driven environment inference; captures are written in source
* **Optimizable** — non-capturing path stays a direct/thin call; capturing path is a known fat callable
* **Layout** — capture struct size and field order are fixed at the creation site
* **Machine code** — no trampolines, no runtime codegen, no hidden heap frames

**Practical rule of thumb**

* Need a scoped helper with a name? → local `fun` (§2.4), pass data as parameters
* Need a callable value that carries data? → lambda with explicit `[...]` capture
* Need to share mutable state with the outer scope? → do not capture; pass `ptr<T>` / mutate in the outer function, or redesign — Bestie will not form a live upvalue

Capturing therefore remains a **lambda (and `bind` / bound-method) feature**, never a silent property of nesting.

---

### 7.4 Explicit Immutable Capture

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

### 7.5 Explicit Mutable Capture

Mutable captures are allowed via `[var x]`. The lambda receives a mutable **copy** that persists between calls of that callable value.

```bestie
var count = 0
var counter = [var count](x: int): int => {
    count += x
    return count
}

counter(5)   // returns 5  — internal count = 5
counter(3)   // returns 8  — internal count = 8
```

The original `count` is **unaffected** — the lambda owns its own mutable copy. This is **not** an upvalue alias.

Rules:

* `[var x]` captures a mutable copy — the original binding is unchanged
* Mutations to `var` captures persist between invocations of that callable
* Cannot be shared across threads — mutable state forbids it
* May be passed, stored, and returned as a light callable value (subject to capture-struct lifetime)
* `own` values cannot be captured as `var`
* No heap allocation — captured state is inline in the callable value / capture struct
* Capture layout is compile-time known

---

### 7.6 Lambda Allocation Model

* Lambdas do not heap-allocate — compile-time lowered to inline code or light callable values
* Immutable captures — part of a zero-size or fixed-size compile-time-known callable context
* Mutable captures — inline in the callable value / capture struct, no heap
* Heap allocation for lambdas is **not part of the core language**
* Environment-style closures (shared outer frames, heap-allocated upvalues) are **not part of the core language**

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

### 10.1 Unbound Method References

```bestie
val f: fn(ptr<User>) -> str = User::getName
```

Equivalent to:

```bestie
(u: ptr<User>) => u.getName()
```

For a value type, the unbound receiver is copied: `fn(Point) -> int`. A `class` is not copyable, so the unbound receiver is `ptr<T>` (or `own T` if the method consumes the instance).

Rules:

* No instance capture
* Instance passed explicitly
* No allocation

---

### 10.2 Bound Method References

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

**`ptr` receivers — unbound, pass explicitly:**

A bound method reference cannot store a `ptr<T>` as an implicit capture of "this object." Use an unbound reference:

```bestie
fun greet(user: ptr<User>) {
    val f: fn(ptr<User>) -> str = User::getName
    f(user)
}
```

---

Rules summary:

* Value types — copied at binding site, no ownership tracking needed
* `own` values — `move` required, source becomes invalid
* `class` via `ptr<T>` — unbound; pass the pointer at the call
* No allocation introduced in any case
* Lowered at compile time

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

* Statically resolved
* Compiled as plain functions
* Desugared at compile time

```bestie
isEmpty(s)
```

No virtual dispatch, no vtables, no runtime lookup.

---

### 11.3 `this` Semantics

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

* Do not use `impl` to satisfy protocols
* Do not participate in protocol dispatch
* Are resolved statically at call site

---

### 11.6 Generic Extensions

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

### 13.1 No Implicit Currying

Automatic currying is not supported.
Capturing-based currying is illegal.

### 13.2 Explicit Partial Application

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

### 16.1 Lambdas Are the Only Anonymous Construct

Bestie has **anonymous functions** (lambdas, section 7) but **no anonymous classes**. There is no inline `impl Protocol { ... }` object expression and no on-the-fly subclass.

Rationale:

* Lambdas already cover the common case — supplying behavior inline — with explicit, zero-cost, non-closure semantics.
* Anonymous classes introduce unnamed types, implicit captures, and fragile-base-class hazards that conflict with Bestie's explicit, named, compile-time model.

When you need an object that implements a protocol, declare a **named class**. For locality, that class can be file-private or an inner class (see `core/oop.md` §8) — it is still named, still statically dispatched, and still has a compile-time-known layout.

### 16.2 SAM Conversion (Lambda → Single-Method Protocol)

A lambda may be supplied wherever a **single-abstract-method (SAM) protocol** is expected. A protocol is SAM when it has **exactly one abstract method** (default methods do not count).

```bestie
protocol Comparator {
    fun compare(a: int, b: int): int
}

fun sortWith(values: list<int>, cmp: Comparator): list<int> { ... }

// A lambda is accepted directly — no named class required.
val sorted = sortWith(values, (a: int, b: int) => a - b)
```

Rules:

* The lambda's parameter and return types must match the SAM method's signature **exactly**.
* The conversion is performed at **compile time**; it synthesizes an implementer with the same allocation and capture semantics as the lambda (no heap, explicit `[...]` captures only).
* Dispatch follows normal protocol rules — static by default.
* Protocols with **two or more** abstract methods are **not** SAM-convertible; implement them with a named class.

---

## 17. Immutability in FP

FP in Bestie strongly prefers immutability.

Guidelines:

* Use `val` by default
* Prefer value types
* Avoid `var` in FP-heavy code
* Favor transformation over mutation

```bestie
val users2 = bestie.lib.functional.map(users, u => u.withName("Alice"))
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
* Environment closures and closure-heavy abstractions (implicit upvalues, shared outer bindings, heap environments, trampolines — see §7.3)
* Anonymous classes (lambdas + named classes cover the same ground explicitly — see section 16)
* Nested methods with implicit outer-`this` capture (local `fun` is unbound — see §2.4)

---

## 19. Summary

Functional programming in Bestie is:

* Explicit
* Compile-time resolvable
* Allocation-aware
* Ownership-safe
* Concurrency-safe
* Zero-cost by design

Local functions give **named locality** without capture. Lambdas give **callable values** with optional **explicit capture-by-copy**. Neither forms a Python-style environment closure.

FP in Bestie exists to **compose behavior clearly**, not to obscure execution.

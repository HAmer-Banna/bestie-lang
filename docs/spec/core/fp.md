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
* Capture only when written in source (a `[...]` list on a lambda — never on local `fun`)
* One callable form per job, named by its type (§6)

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
    for (item in items) {
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
* A local function may not shadow another local function in the same block; shadowing an outer local function across nested blocks follows normal scope shadowing (`lang.md` §4.8)

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

Local functions **do not support capture lists**. Capturing is exclusively the domain of lambdas (§7). If a named capturing callable is needed:

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
* A local function converts to every callable form a named function does (§6.2): `fn(T) -> R`, `fn[](T) -> R`, `foreign fn(T) -> R` when C-representable, and a generic `F`
* No trampolines, no heap frame, no runtime code generation
* No hidden allocation

#### Local function vs lambda

| | Local `fun` | Lambda |
| --- | --- | --- |
| Name | Named, recursive by name | Anonymous (bind to `val`/`var` for a name) |
| Capture | Never — pass parameters | None by default; explicit `[...]` list (§7.5–§7.8) |
| Role | Scoped static helper | Inline / first-class behavior value |
| Lowering | Mangled static function | A value of its own type holding its captures (§6.2) |

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

There is no named `option<T>` type and no constructors to import — `T ?` is the only spelling. In expression position the absent value is written `absent` (`types.md` §8.3); presence needs no constructor, since a `T` is accepted wherever a `T ?` is expected.

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

**Both cases — core.** `if`-let with an `else` covers the two-branch form:

```bestie
if (val user = getUser(id)) {
    sendEmail(user)
} else {
    println("user not found")
}
```

There is no `switch` form for `T ?`, because there is nothing a two-arm `switch` would add over `if`-let — and no constructors to name in a `case` pattern.

**`while`-let — core.** Repeat while the expression is present. The expression is evaluated again before every pass, and the loop ends at the first absent value:

```bestie
while (val job = queue.poll()) {
    run(job)
}
```

`while`-let has no `else`. `break` and `continue` work as in any `while` (`lang.md` §13).

**Binding a place borrows it.** When the expression is a call, `val x` holds the result. When it is a **place** — a field, an element, or a binding — what `x` holds depends on the type:

| Inner type | `if (val x = place)` / `while (val x = place)` |
| ---------- | ---------------------------------------------- |
| Copyable (`int`, `value class`, `ptr<T>`, …) | A copy, as with any `val` |
| `own T` or a `class` | A **borrow** of the value in the place, valid for the block |

A borrow can be read and mutated through, but inside the block it cannot be moved, freed, or stored, and the place it came from cannot be assigned, moved out of, or swapped. The check is local to the block, so it needs no lifetime analysis. Looking inside an optional field never empties it by accident. To take ownership, say so with `move`:

```bestie
if (val n = this.top) { n.value += 1 }            // borrow; top is unchanged
if (val own n = move this.top) { n.free() }       // move; top is now absent
```

Compiler enforces:

* `T ?` cannot be used directly where `T` is expected — unwrapping is required
* `if`-let binds the inner `T` value — the outer `T ?` is not accessible inside the block
* An absent value reaching a context that requires `T` is a compile-time error, never a runtime fault

### 3.4 Lambdas and Partiality

Lambdas may also be complete or partial.

Complete lambda:

```bestie
val inc = (x: int) => x + 1
```

Partial lambda:

```bestie
val findUser = (x: int): User ? => {
    if (x > 0) { return repo.get(x) }
    return
}
```

### 3.5 Absence in Data Structures — Where Java Writes `null`

Java uses `null` for three different jobs, all spelled `x = null`. Bestie gives each job its own spelling:

| Job | Java | Bestie |
| --- | ---- | ------ |
| This slot may be empty | `Node next;` … `next = null` | The type says so: `var own next: Node ?`, emptied with `absent` |
| Take the value out and leave the slot empty | `Node n = head; head = null;` | `val own n = move this.head` — the place becomes `absent` (`memory.md` §5.2.1) |
| This sequence holds fewer than its capacity | `arr[i] = null`, then skip nulls | Keep a count and view `arr[..count]`, or use `bounded<T, N>` (`std-lib/collections.md` §2b) |

**The end of a linked list** is an absent link. `own Node ?` costs nothing over C: the zero address is a niche of `own T`, so the field is one pointer, and the compiler tests it exactly as C tests `NULL` (`memory-layout.md` §11). The program, though, can never read that zero address:

```bestie
class Node {
    var value: int
    var own next: Node ?             // absent = end of list
}

class IntList {
    var own head: Node ?

    fun pushFront(v: int) {
        val own n = Node.new(value: v, next: move this.head)
        this.head = move n           // head was emptied by the move above
    }

    fun sum(): int {
        var total = 0
        var cur = this.head.address()                // ptr<Node> ? — memory.md §8.3
        while (val n = cur) {
            total += n.val.value
            cur = n.val.next.address()
        }
        return total
    }

    fun clear() {
        while (val own n = move this.head) {
            this.head = move n.next
            n.free()
        }
    }
}
```

**An array slot that may be empty** is `array<T ?>[N]`, emptied with `slots[i] = absent`. When `T` has a niche — a `class` reference, `bool`, an `enum`, a range-constrained integer — `T ?` is the same size as `T`. When it has none, as with a plain `int`, every element gains a tag byte; that cost shows in the type, where `?` is written.

**An array that is "partly empty"** is usually a sequence shorter than its storage, and should not be modeled with empty slots at all. Keep a count beside the array and pass `arr[..count]`. This is the C idiom: no flags to scan, no gaps, and every slice and algorithm works on the live part. `bounded<T, N>` in `bestie.lib.collections` packages exactly that.

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
| `T?` (nullable reference) | `T ?` (core syntax — one spelling, no named alias) |
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

A library written in Bestie is structurally incapable of introducing null. It can only express absence via `T ?`. This is enforced by the type system — not a convention, not a guideline.

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

## 6. Function Types and Callables

A **callable** is anything that can be called with `f(args)`. Bestie has one form of callable for each job, and the type says which one you have. An expert C programmer uses the same four things: a function pointer, a template parameter, a `(callback, void*)` pair, and a function pointer with a C calling convention.

| Form | Size | Carries captures | Storable | Call | Job |
| ---- | ---- | ---------------- | -------- | ---- | --- |
| `fn(T) -> R` | 1 word | No | Anywhere | Indirect | A function pointer: dispatch tables, hand-built vtables, plugin entry points, interpreters (§6.3) |
| `<F impl fn(T) -> R>` | The captures, inline | Yes, by value | Yes, as `F` | Direct, inlinable | Higher-order parameters and generic fields: `map`, `sort`, a widget's handler (§6.5) |
| `fn[](T) -> R` | 2 words | Yes, by address | No — a view, like `slice<T>` | Indirect | A non-generic parameter that accepts any callable, including capturing lambdas; synchronous C callbacks (§6.4) |
| `foreign fn(T) -> R` | 1 word | No | Anywhere | Indirect, C ABI | A C function pointer (§6.6, `memory.md` §8.9) |

A heap-owned callable that can be stored in a field or a collection and carries captures — Kotlin's `(T) -> R`, C++'s `std::function` — is **not** a core type. It costs an allocation, so it lives in `bestie.lib.functional` as `Closure<fn(T) -> R>` (`std-lib/functional.md`), built from the generic form, the function pointer `fn`, and `ptr<T>` (`vision.md` Rule 4). Creating one is a named `.new()`.

### 6.1 The `fn` Prefix Is Required

A function type is **always** written with the `fn` prefix and a single arrow `->` from the parameter list to the return type. There is no bare form:

```bestie
val f: fn(int) -> int = square      // ✅
val g: (int) -> int   = square      // ❌ function types are spelled with 'fn'
```

Three shapes use parentheses, and `fn` plus the arrow distinguishes all of them at a glance:

* A **function type** — `fn(int) -> int`
* A **lambda** (a value) — `(x: int) => x * 2`
* A **tuple type** — `(int, str)`

The arrows alone would be enough to parse, which is why this prefix was once optional. Optional is the wrong setting: a marker that half the code carries is a style argument, not a signal, and Bestie keeps **one spelling per idea** (§11.4 of `oop.md` makes the same call about header constructors). `fn` is part of the type, so it is greppable, and every callable form is built from it:

| Spelling | Reads as |
| -------- | -------- |
| `fn(int) -> int` | a function pointer |
| `fn[](int) -> int` | a function with captures — the `[]` echoes the capture list `[x](…) => …` of a lambda |
| `foreign fn(int32) -> void` | a function with the C calling convention — `foreign` is the keyword of `foreign fun` |
| `F impl fn(int) -> int` | anything callable with this shape — `impl` is the generic-constraint keyword (`lang.md` §7.1) |

`fn` is a contextual keyword (`lang.md` §3.1.4): it has meaning only at the head of a function type, so `val fn = 3` remains legal.

### 6.2 Every Function and Lambda Has Its Own Type

Each named function, local function, and lambda expression has its own **anonymous callable type**, created by the compiler. The type of a lambda holds exactly its captures (§7), in declaration order, so its size is the size of its captures — zero for a non-capturing lambda or a named function.

```bestie
val double = (x: int) => x * 2         // its own type, 0 bytes
val k = 3
val scale = [k](x: int) => x * k       // its own type, 8 bytes: one int
```

The anonymous type cannot be written in source. It is what `val` infers, what a generic `F` is bound to (§6.5), and what converts to the other forms where one is expected:

| From | → `fn(T) -> R` | → `fn[](T) -> R` | → `foreign fn(T) -> R` | Binds `F impl fn(T) -> R` |
| ---- | -------------- | ---------------- | ---------------------- | ------------------------- |
| Named or local function | ✅ | ✅ | ✅ if the signature is C-representable | ✅ |
| Non-capturing lambda | ✅ | ✅ | ✅ if the signature is C-representable | ✅ |
| Capturing lambda | ❌ it carries data | ✅ | ❌ | ✅ |
| A type that `impl fn(T) -> R` (§6.7) | ❌ | ✅ | ❌ | ✅ |
| `fn(T) -> R` | — | ✅ | ❌ | ✅ |
| `foreign fn(T) -> R` | ❌ | ✅ | — | ✅ |

Every conversion is implicit, because none of them allocates or copies anything but a word: each is a compile-time choice of representation. The one conversion that allocates — into a `Closure` — is a call to `.new()`.

### 6.3 `fn(T) -> R` — The Function Pointer

`fn(T) -> R` is **one word: a code address**, nothing else. It is exactly a C function pointer, with Bestie's calling convention.

```bestie
val ops: array<fn(int, int) -> int>[] = { add, sub, mul, div }   // 4 words in .rodata
val r = ops[opcode](a, b)                                        // one indirect call
```

Rules:

* Only callables with no captures convert to `fn(T) -> R` (§6.2): named functions, local functions, and non-capturing lambdas.
* A `fn(T) -> R` may be stored in a field, an array, a collection, or a module-level `val`, returned, and sent across threads. It points at code, which is never freed.
* It is never the zero address. An optional function pointer is `fn(T) -> R ?`, still one word: the zero address is the niche (`memory-layout.md` §11).
* `==` and `!=` compare code addresses. `f.addr` is the address word as a `uint`, as on `ptr<T>` (`memory.md` §8.4.2).
* The address of a named function is a **link-time constant**, so a table of them is a valid module-level initializer (`freestanding.md` §6.2).

### 6.4 `fn[](T) -> R` — The Callable View

`fn[](T) -> R` is **two words: a code address and a context address**. It is the C idiom of a function pointer plus a `void*` user-data argument, with the types checked. Any callable converts to it (§6.2), so it is the parameter type for a function that wants to accept capturing lambdas without being generic.

```bestie
fun forEachLine(text: str, visit: fn[](str) -> void) { ... }

val prefix = "> "
forEachLine(doc, [prefix](line) => print("${prefix}${line}"))
```

The context word is the **address of the callable value itself** — the lambda's captures, wherever they already live. Converting a capturing lambda to `fn[]` copies nothing: for a lambda written in an argument list, the captures are a temporary in the caller's frame; for a lambda bound to a local, they are that local.

Because the context points into the caller's storage, `fn[]` follows exactly the **view rules of `slice<T>`** (`types.md` §6, `memory.md` §13):

* It **cannot be stored** in a field, a collection element, or a module-level binding.
* It **cannot outlive** the callable it views. It may be passed into calls and bound to locals; it cannot be returned, except a `fn[]` parameter returned unchanged.
* It cannot be sent across threads.
* `fn[](T) -> R ?` is still two words: the code word's zero address is the niche.

These are the same checks the compiler already performs for slices, so `fn[]` adds no new analysis. To keep a capturing callable beyond the call, store it as a generic `F` (§6.5) or allocate a `Closure` (`std-lib/functional.md`).

### 6.5 Generic Callables — `<F impl fn(T) -> R>`

Inside a generic parameter list, `fn(T) -> R` is also a **constraint**: `F impl fn(T) -> R` accepts any callable of that shape — a named function, any lambda, a `fn`, a `fn[]`, a `foreign fn`, or a type that implements `fn(T) -> R` (§6.7). The function is monomorphized for each distinct `F`, exactly like any other generic parameter (`lang.md` §7.1).

```bestie
fun <C impl Iterable<T>, T, R, F impl fn(T) -> R> map(xs: C, f: F): list<R>

val ys = map(nums, x => x * 2)
```

This is the form that compiles to what a C programmer writes by hand. `F` is the lambda's own type, so the call `f(x)` inside `map` is a **direct call to known code**, and the inliner sees `x * 2` in the loop. A non-capturing `F` is zero bytes and passing it costs nothing; a capturing `F` is passed as its captures, like any value.

**Storing a generic callable.** A field of type `F` holds the callable's captures inline. There is no indirection and no allocation:

```bestie
class Button<F impl fn() -> void> {
    val label: str
    val onClick: F
}

val count = counter.address()
val own b = Button.new(label: "OK", onClick: [count]() => count.val.increment())
b.onClick()                  // direct call; the capture lives inside b
```

**Code size.** Each distinct callable type gives one instantiation — one per lambda, exactly as in a C++ template. That is linear in call sites, never combinatorial (`vision.md` Rule 5). Where code size matters more than the call, take `fn[](T) -> R` instead: one copy of the function, one indirect call per invocation.

**Which to use for a parameter:**

| Parameter | Choose when |
| --------- | ----------- |
| `f: F` with `F impl fn(T) -> R` | The default. The callable runs in a loop or a hot path, or it is stored |
| `f: fn[](T) -> R` | The function should not be generic — one compiled copy, a stable signature, a large body — and the callable is used only during the call |
| `f: fn(T) -> R` | Only plain functions make sense, and the value is stored as a pointer: handlers in a table, a signal handler, an entry point |

### 6.6 `foreign fn(T) -> R` — C Function Pointers

`foreign fn(T) -> R` is one word, a code address with the **C calling convention**. It is the type of every C function pointer: a callback a C library takes, a symbol loaded at run time, an entry in a vector table. Its parameter and return types must be C-representable (`std-api/foreign.md` §5).

```bestie
foreign fun atexit(handler: foreign fn() -> void): int32

fun flushLogs() { ... }
atexit(flushLogs)          // the compiler emits flushLogs with the C convention
```

A named function, local function, or non-capturing lambda converts to `foreign fn` when its signature is C-representable; the compiler emits a C-ABI entry point for it (no exported symbol is created). A `foreign fn` is called directly, `cb(x)`, with the trust of any `foreign fun` call. A C function pointer that may be `NULL` is `foreign fn(T) -> R ?` (`std-api/foreign.md` §8). The raw-address operations are in `memory.md` §8.9; callbacks with a user-data argument are in `std-api/foreign.md` §9.

### 6.7 Types That Implement `fn(T) -> R`

A `value class` or `class` may implement a function shape by declaring `impl fn(T) -> R` and a method named `call` with that signature. A call expression on such a value lowers to the method, statically, like every operator (`lang.md` §15.3):

```bestie
value class Adder impl fn(int) -> int {
    val n: int
    fun call(x: int): int = x + this.n
}

val add5 = Adder.new(n: 5)
add5(10)                     // add5.call(10) — 15
```

This is how a library builds a callable with a name: `Closure<fn(T) -> R>` and the result of `compose` (§12) are such types. A type may implement more than one function shape, as long as no call is ambiguous.

### 6.8 Lowering and Calling Convention

| Callable | Representation | Call instruction |
| -------- | -------------- | ---------------- |
| Named function, statically known | none | direct `call` (or inline) |
| Any lambda bound to a generic `F` | its captures (0 bytes if none) | direct `call` (or inline) |
| `fn(T) -> R` | 1 word — code | indirect `call` |
| `fn[](T) -> R` | 2 words — code, context | indirect `call`, context in a hidden register |
| `foreign fn(T) -> R` | 1 word — code | indirect `call`, C convention |

**The context register.** A call through `fn[]` passes the context word in a register that Bestie's calling convention reserves for it and that is not an argument register. A function with no context never reads it. So the code word of a `fn[]` made from a named function is that function's own address — there is no adapter, no thunk, and **no branch on whether a context is present**. A `fn[]` made from a `foreign fn` uses one compiler-emitted adapter per signature, with the `foreign fn` itself as the context word.

There are no trampolines on the stack, no executable stacks, no run-time code generation, and no hidden allocation in any row.

---

## 7. Lambdas

Lambdas are anonymous functions with **explicit and restricted semantics**.

### 7.1 Lambda Syntax

```bestie
val f = (x: int): int => x * 2
val g = (x: int) => x * 2          // return type inferred from the body
```

A block body uses braces and `return`:

```bestie
val clamp = (x: int) => {
    if (x < 0) { return 0 }
    return x
}
```

Properties:

* Return type is inferred from the body, or written after the parameter list
* No implicit allocation
* Compile-time lowered — each lambda has its own type (§6.2)

### 7.2 Inferred Parameter Types

Where the lambda is written against an **expected function shape** — an argument to a parameter of type `fn(T) -> R`, `fn[](T) -> R`, `foreign fn(T) -> R`, or `F impl fn(T) -> R`, or a binding with a declared function type — the parameter types may be left out. A single untyped parameter needs no parentheses:

```bestie
val ys = map(nums, x => x * 2)
sort(people[..], (a, b) => a.age < b.age)
val h: fn(int) -> int = x => x + 1
```

Rules:

* Types come from the expected shape at that one site — the same single, dependency-ordered pass that infers generic parameters (`lang.md` §7.1). There is no inference from how the lambda is later used, and no inference across statements.
* For a generic `F impl fn(T) -> R`, `T` must already be known from earlier parameters when the lambda is reached; `R` is then inferred from the body.
* With no expected shape — `val f = x => x * 2` — the types are required. That is a compile-time error, not a guess.
* Writing some parameter types and omitting others in one lambda is not allowed: either all are written or none are.

### 7.3 Non-Closure Rule (Core Guarantee)

**Bestie does not have environment closures.**

In Python, JavaScript, Kotlin, and similar languages, a nested function or lambda silently forms a *closure* over its lexical environment: free variables resolve to the outer frame (often by reference), lifetimes extend with the callable, and allocation / sharing costs are hidden.

Bestie rejects that model. By default, lambdas and local functions:

* Capture nothing
* Access only:

  * Their parameters
  * Compile-time constants (`const`)
  * Module-visible functions, types, and module-level bindings

Illegal without an explicit capture list:

```bestie
val y = 10
val f = (x: int) => x + y   // compile-time error — y is not captured
val g = [y](x: int) => x + y   // ✅
```

Guarantees:

* No hidden sharing
* No outer-frame / upvalue machinery
* No lifetime complexity beyond ordinary scope rules
* Zero allocation

### 7.4 Closures in Bestie — Evaluation and Position

Bestie’s position is deliberate and locked to the platform pillars (`docs/platform.md`):

| Model | Status in Bestie |
| --- | --- |
| Implicit lexical closures (Python / JS / Kotlin default) | **Rejected** |
| Closures that share / mutate the outer binding by reference | **Rejected** — capture an address explicitly (§7.8) |
| Closures that heap-allocate an environment object | **Rejected** in core — `Closure.new` in `bestie.lib.functional`, named at the call |
| Escaping nested functions via trampolines / executable stacks | **Rejected** |
| Explicit capture lists on lambdas (`[x]`, `[var x]`, `[move x]`, `[name = expr]`) | **Allowed** — the only capturing form |
| Local `fun` with free outer locals | **Rejected** — pass parameters (§2.4) |

What Bestie *does* provide is not an environment closure. It is an **explicit capture callable**:

* Every captured name appears in a `[...]` list at the creation site
* Every capture is **a value** — a copy, a moved owner, or an address the programmer took — stored in the lambda's own type (§6.2)
* `[var x]` still copies — it creates **private mutable state inside the callable**, not a live alias to the outer `var`
* The lambda value is an ordinary value: where it lives, and how long, is visible in the code that holds it

```bestie
var count = 0
var counter = [var count](x: int): int => {
    count += x
    return count
}
count = 100
counter(1)   // returns 1 — outer `count` was never shared
```

**Why this stays pillar-safe**

* **Compilation speed** — no escape-driven environment inference; captures are written in source
* **Optimizable** — a lambda bound to a generic `F` is a direct call to known code (§6.5)
* **Layout** — the lambda's size and field order are fixed at the creation site
* **Machine code** — no trampolines, no runtime codegen, no hidden heap frames

**Practical rule of thumb**

* Need a scoped helper with a name? → local `fun` (§2.4), pass data as parameters
* Need a callable value that carries data? → lambda with an explicit `[...]` capture
* Need the callable to change outer state? → capture its address: `[total = total.address()]` (§7.8)
* Need to keep it beyond the current call? → a generic field `F` (§6.5), or `Closure.new` when callables of different types share one slot

Capturing therefore remains a **lambda feature**, never a silent property of nesting.

### 7.5 Copy Capture — `[x]`

```bestie
val factor: int = 3
val f = [factor](x: int) => x * factor
```

Rules:

* The value is copied into the lambda at the point of creation
* The capture is immutable — it cannot be reassigned inside the lambda
* Only copyable types can be copy-captured (`memory.md` §6.1). A `class` cannot: capture it with `[move x]` or by address (`[x = x.address()]`)

### 7.6 Mutable Capture — `[var x]`

`[var x]` captures a mutable **copy** that persists between calls of that callable value.

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
* Mutations persist between invocations of that callable value
* Calling a lambda with a `[var]` capture mutates it, so it must be held in a `var` binding or a `var` field, and it converts to `fn[]` only from a `var` place
* It cannot be sent across threads while another thread can call it

### 7.7 Move Capture — `[move x]`

`[move x]` moves an `own` value into the lambda. The outer binding is invalid afterward, exactly as after any `move` (`memory.md` §5).

```bestie
val own conn = try Connection.open(addr)
val own send = [move conn](msg: str) => conn.write(msg)
send("hello")
send("world")
send.freeDeep()              // frees conn with the lambda
```

Rules:

* The lambda now holds an ownership obligation, so it is itself an `own` value: it is not copyable, it is moved like any owner, and it is discharged with `freeDeep()`, which frees its moved captures in reverse capture order (`memory.md` §7.2). Forgetting to discharge it is the ordinary leak error.
* Inside the body, a moved capture is a **borrow** of the lambda's field — readable and mutable through, but it cannot be moved out, freed, or stored. The rule is the one for binding a place in `if`-let (§3.3).
* A callable with a moved capture binds a generic `F` (`own f: F`) or a `Closure`, and converts to `fn[]` like any callable. It never converts to `fn` or `foreign fn`.
* To hand the value away from inside the body, capture it as `[var move x]` with type `own T ?`, and `move` out of it — leaving it `absent`, as with any optional place (§3.5).

### 7.8 Initialized Capture — `[name = expr]`

A capture may give a new name to the value of any expression, evaluated once when the lambda is created:

```bestie
var total = 0
val tally = [t = total.address()](x: int) => t.val += x     // captures a ptr<int>
for (x in xs) { tally(x) }                                    // total now holds the sum

val self = [s = this.address()]() => s.val.refresh()          // the receiver of a method
val first = [n = names[0]]() => greet(n)                      // one element, copied
```

Rules:

* `[name = expr]` copies the result; `[var name = expr]` makes the copy mutable; `[move name = expr]` moves an `own` result in.
* An initialized capture of `x.address()` is how a lambda reaches outer state. The pointer is a `ptr<T>`, with the same programmer-owned lifetime as any pointer (`memory.md` §13). Nothing is aliased that the source does not show.

### 7.9 Lambda Allocation Model

* Lambdas never heap-allocate. A lambda value is its captures, laid out like a `value class` with those fields.
* Where the value lives is where it is bound: a local, a field, a temporary in an argument list.
* Heap allocation for callables is a named library call, `Closure<…>.new(…)` (`std-lib/functional.md`).
* Environment-style closures (shared outer frames, heap-allocated upvalues) are **not part of the core language**.

---

## 8. Higher-Order Functions

A higher-order function takes its callable as a generic parameter by default (§6.5):

```bestie
fun <F impl fn(int) -> int> apply(f: F, x: int): int {
    return f(x)
}

apply(x => x + 1, 41)                 // direct call, inlined
```

When the function should not be generic, it takes a view:

```bestie
fun retry(times: int, attempt: fn[]() -> bool): bool {
    for (i in 0..times) {
        if (attempt()) { return true }
    }
    return false
}
```

Rules:

* Function arguments are explicit
* No runtime boxing, and no allocation in any form
* A generic callable parameter is a direct call; a `fn[]` or `fn` parameter is one indirect call
* Fully compatible with ownership and concurrency rules

---

## 9. Variable Arguments (Varargs)

Variadic parameters are declared with `...` after the element type.
`val` and `var` carry their normal semantics — `val` is the default and preferred form.

```bestie
fun sum(val xs: int...): int {
    var total = 0
    for (x in xs) {
        total += x
    }
    return total
}
```

Mutable varargs allow modification of elements inside the function body:

```bestie
fun normalize(var xs: float64...): void {
    for (i in 0..xs.length) {
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

* A method reference has no captures, so it converts to every callable form (§6.2), including `fn` and — for a C-representable signature — `foreign fn`
* The instance is passed explicitly at the call
* No allocation

---

### 10.2 No Bound Method References

There is no `p::getX` form that binds an instance into a callable. A method reference is always **unbound**: the receiver is passed at the call.

To carry a receiver, write a lambda with an explicit capture — a copy for a value type, an address for a `class`:

```bestie
val p = Point.new(x: 1, y: 2)
val f = [p]() => p.getX()                         // p copied into the lambda

val g = [s = server.address()]() => s.val.stop()  // server is a class: capture its address
```

A bound reference would have been a second way to build a capturing callable, with its own rule for value receivers, `class` receivers, and `own` receivers. The capture list already says what is carried and how, at the site where it happens.

---

## 11. Extension Functions

Extension functions add behavior **without modifying types** and **without runtime cost**.

### 11.1 Declaration

```bestie
fun str.isEmpty(): bool {
    return this.byteSize() == 0
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

A composed function carries the two functions it calls, so it is a value with captures. It cannot be returned as a `fn[]` (a view cannot outlive what it views, §6.4), and a lambda's type cannot be named in a return type. The library therefore returns a named type that implements the composed shape (§6.7):

```bestie
value class Composed<A, B, C, F impl fn(B) -> C, G impl fn(A) -> B> impl fn(A) -> C {
    val f: F
    val g: G
    fun call(x: A): C = this.f(this.g(x))
}

fun <A, B, C, F impl fn(B) -> C, G impl fn(A) -> B> compose(f: F, g: G): Composed<A, B, C, F, G> =
    Composed.new(f: f, g: g)
```

`compose(f, g)(x)` is two direct calls, inlined. `compose` is declared in `bestie.lib.functional` (`std-lib/functional.md`); core provides only the pieces. No implicit currying or composition exists.

---

## 13. Currying and Partial Application

Neither is a language feature. There is no implicit currying, and there is no `bind`.

Partial application is a lambda with an explicit capture list:

```bestie
val n = getUserInput()
val addN = [n](b: int) => add(n, b)     // n copied into the lambda
addN(3)
```

A `bind(...)` form would have been a second spelling for exactly this, with the capture hidden behind a call instead of written in a `[...]` list. The lambda states its captures at the creation site, which is the property §7.4 exists to protect, and it specializes identically when the captured value is a compile-time constant.

To keep a partially applied function beyond the current scope, store the lambda in a generic field (§6.5) or allocate a `Closure` (`std-lib/functional.md`).

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

* `own T` cannot cross boundaries implicitly
* Ownership passing is explicit — including into a lambda, with `[move x]` (§7.7)
* A lambda that holds a moved capture is itself an owner and is discharged with `freeDeep()`
* `fn[]` follows the view rules of `slice<T>`: it cannot be stored and cannot outlive what it views (§6.4)

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

* Lambdas already cover the common case — supplying behavior inline — with explicit, zero-cost, non-closure semantics. A named type that implements a function shape (§6.7) covers the rest.
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

* The lambda's parameter and return types must match the SAM method's signature **exactly**. Untyped lambda parameters take the method's types (§7.2).
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
val users2 = bestie.lib.functional.map(users, u => u.withName("Alice"))   // u: User, from map's signature (§7.2)
```

Enforced by:

* Type system
* Ownership rules
* Compile-time checks

---

## 18. What Bestie Deliberately Avoids in FP

* Implicit currying and any `bind` / partial-application form (§13)
* Bound method references (§10.2)
* A heap-allocated callable type in core — `Closure` is a library type with a named `.new()` (§6)
* Trailing-lambda call syntax — a lambda is an ordinary argument
* Lazy evaluation by default
* Runtime monads
* Hidden effect systems
* Reflection-based dispatch
* Environment closures and closure-heavy abstractions (implicit upvalues, shared outer bindings, heap environments, trampolines — see §7.4)
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

Local functions give **named locality** without capture. Lambdas give **callable values** with an explicit capture list — copy, mutable copy, move, or an initialized name. Neither forms a Python-style environment closure.

Callables come in one form per job:

| Form | Job |
| ---- | --- |
| `fn(T) -> R` | Function pointer — one word, storable |
| `<F impl fn(T) -> R>` | Generic callable — direct call, captures inline, storable as `F` |
| `fn[](T) -> R` | Callable view — any callable, two words, not storable |
| `foreign fn(T) -> R` | C function pointer |
| `Closure<fn(T) -> R>` (library) | Heap-owned callable, created with a named `.new()` |

FP in Bestie exists to **compose behavior clearly**, not to obscure execution.

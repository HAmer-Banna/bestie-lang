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
* Lifetime and destructor placement
* Ownership validation
* Pointer correctness
* Builder chains
* Error handling paths
* Loop lowering and unrolling opportunities
* Devirtualization
* Concurrency safety for ownership/sharing rules (`own/ref`)

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

The core defines **semantic guarantees**.
Higher layers provide **convenience and extensibility**.

---

## 4. Variables and Bindings

Bestie provides three binding forms.

### 4.1 `const` — Compile-Time Constant

`const` defines an immutable binding and immutable value resolved entirely at compile time.

Properties:

* Immutable binding and value
* No runtime allocation
* No mutation through references
* Cannot reference mutable or runtime memory
* Stored in read-only memory

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

`val` defines an immutable binding to a value.

The **binding is immutable**.
The **value mutability depends on its type**.

Properties:

* Runtime values allowed
* Preferred default
* Encourages value-oriented programming

File-level `val` must be annotated with `@immutable`.

---

### 4.3 `var` — Mutable Binding (Restricted)

`var` defines a mutable binding and is deliberately restricted.

Allowed only:

* Local variables
* Class properties with explicit accessors

Disallowed:

* File scope
* Protocols
* `data`, `value`, and `enum` declarations

---

### 4.4 Multiple Value Declarations

Multiple value declarations are **binding syntax sugar**, not data structures.

```bestie
val x, y, z = 5, 6, 3
```

Rules:

* RHS values must share the **same type**
* No tuple is created
* Binding is positional
* Lowered to independent bindings at compile time
* Zero runtime cost

---

### 4.5 Tuples and Destructuring

Tuples are **first-class heterogeneous value types**.

```bestie
val (x, y, z) = (5, 6, 'c')
```

Properties:

* Heterogeneous allowed
* Compile-time layout known
* No implicit allocation
* Explicit tuple semantics via parentheses

---

### 4.6 Ignoring Values with `_`

`_` explicitly discards values without creating bindings.

Properties:

* No lifetime
* No storage
* Compile-time only
* Suppresses unused diagnostics

---

## 5. Types

### 5.1 Primitive Types

Primitive value types map directly to machine types:

| Type                              | Width         | Notes                                    |
| --------------------------------- | ------------- | ---------------------------------------- |
| `int8`, `int16`, `int32`, `int64` | Fixed         | Signed integers, explicit width          |
| `uint8`, `uint16`, `uint32`, `uint64` | Fixed     | Unsigned integers, explicit width        |
| `byte`                            | 8-bit         | Alias for `uint8`. Used for raw memory.  |
| `int`                             | Pointer-sized | Signed. Equivalent to C `intptr_t`. Use for indices, sizes, offsets. |
| `uint`                            | Pointer-sized | Unsigned. Equivalent to C `uintptr_t` / `size_t`. |
| `float32`, `float64`              | 32 / 64-bit   | IEEE 754                                 |
| `bool`                            | 1-bit logical | `true` / `false`                         |
| `char`                            | 32-bit        | Unicode scalar value (valid codepoint)   |

`double` does not exist. Use `float64`.

**`int` and `uint` — pointer-sized by design:**

`int` and `uint` are not ambiguous — their width is the pointer width of the target platform. They exist specifically for indices, sizes, counts, and pointer arithmetic. Using `int32` or `int64` is correct when the bit width matters independently of the platform.

Mixing `int` with fixed-width types (`int32`, `int64`, etc.) requires an explicit cast. No silent narrowing or widening.

**`byte` and `uint8`:**

`byte` is an alias for `uint8`. They are the same type — no cast required between them. `byte` is the idiomatic name when working with raw memory, buffers, and binary data. `uint8` is idiomatic when the numeric value matters.

All primitives:

* No headers
* Stack-friendly
* No deallocation
* Deterministic layout

---

### 5.2 Core Value Types

Includes:

* `str` (immutable, UTF-8)
* `tuple`
* `ptr<T>`

**`str` and `char` indexing:**

`str` is UTF-8 encoded. Two indexing modes exist to separate speed from correctness:

| Operation        | Returns | Meaning                                      |
| ---------------- | ------- | -------------------------------------------- |
| `s[i]`           | `byte`  | Raw UTF-8 byte at byte offset `i`. C-speed.  |
| `s.char(i)`      | `char`  | Unicode scalar at codepoint index `i`.       |
| `s.chars()`      | iterator over `char` | Full Unicode iteration.         |

`s[i]` is direct memory access — fast, no validation. Use it when you are working with bytes.
`s.char(i)` is the human-readable path — validates and decodes. Use it when you care about characters.

No hidden cost. The fast path stays fast. The correct path is explicit.

Properties:

* Binding immutability defaults to `val`/`const`
* Value mutability depends on type qualifiers
* Efficient copy semantics
* No hidden allocation

---

### 5.3 Core Collections

Bestie includes exactly one collection in the core language: `list<T>`.

`list<T>` is built-in and available without `import`.
It participates directly in the language's type, memory, and loop rules.

Core `list<T>` supports:

* `list<T>.build()` with no import
* Built-in variations such as `array` and `linked`
* Indexing via `xs[i]`
* List literals such as `{1,2,3}` when the target type is `list<T>`
* Sized array forms such as `list<int>[10]` and `list<int>[2][3]`
* Core methods such as `add`, `remove`, `get`, `insert`, and `indexOf`
* Ownership, immutability, and concurrency semantics defined by the core specification
* Builder-chain resolution at compile time

Bestie does **not** provide a separate built-in array type.
Array semantics are expressed through the core `list<T>`.
The default `list<T>` is array-backed unless another core variation is selected.

All other collections live in `bestie.lib.collections`, including:

* `set<T>`
* `map<K,V>`
* `deque<T>`
* `heap<T>`

Example:

```bestie
val xs = list<int>.linked().build()
```

---

## 6. Casting and Type Rules

Bestie requires **explicit casting**.

No implicit numeric promotion or narrowing is allowed.

---

### 6.1 Type Aliases — Newtypes (`type X as Y`)

`type X as Y` declares a **newtype**: a distinct type with the same underlying representation as `Y`.

```bestie
type Meters as float64
type Seconds as float64
```

**Properties:**

* `X` is a distinct type from `Y` — they cannot be used interchangeably
* No implicit conversion in either direction
* Zero runtime overhead — same memory layout as `Y`
* `X` automatically inherits all methods and operators of `Y`
* `X` may implement additional protocols

**Casting:**

```bestie
val m: Meters  = 10.0 as Meters    // wrap
val raw: float64 = m as float64    // unwrap
```

**Type safety:**

```bestie
val m: Meters  = 10.0 as Meters
val s: Seconds = 5.0 as Seconds

val bad = m + s    // ❌ compile error: Meters and Seconds are distinct types
val good = m + m   // ✅ Meters inherits float64's + operator
```

**If full interchangeability is needed**, use the original type directly. `type X as Y` is always a newtype — there is no alias-without-distinction form.

---

## 7. Generics

Generics are **compile-time only**.

Properties:

* Full monomorphization
* No runtime type information
* No type erasure
* Predictable layout and performance

---

### 7.1 Generic Constraints

Generic type parameters may be constrained using `ext` and `impl`, consistent with the rest of the type system.

**Protocol constraint:**

```bestie
fun <T impl Comparable> sort(xs: list<T>): list<T>
fun <T impl Hashable> dedupe(xs: list<T>): list<T>
```

**Class constraint:**

```bestie
fun <T ext Shape> render(t: T): void
```

**Multiple constraints of the same kind** — comma-separated, no need to repeat the keyword:

```bestie
fun <T impl Comparable, Hashable> index(t: T): int
```

**Mixed constraints** — restart with the new keyword:

```bestie
fun <T ext Animal impl Printable, Serializable> describe(t: T): str
```

**Multiple type parameters:**

```bestie
fun <K impl Hashable, V> lookup(m: map<K, V>, key: K): V ?
```

**Constraints on types and protocols:**

```bestie
data class Pair<T impl Equable> {
    first: T
    second: T
}

protocol Container<T impl Hashable> {
    fun contains(item: T): bool
}
```

Rules:

* `T ext X` — `T` must be a subclass of `X` (`X` must be `open` or `abstract`)
* `T impl P` — `T` must implement protocol `P`
* Comma continues the current `impl` or `ext` group
* A new `impl` / `ext` keyword starts a new group
* All constraints are AND semantics — every constraint must be satisfied
* Violations are **compile-time errors**
* No runtime dispatch introduced — still fully monomorphized

---

## 8. Numeric Literals

Bestie supports explicit numeric literal forms for systems-level clarity.

### 8.1 Integer Literals

| Form              | Example        | Notes                              |
| ----------------- | -------------- | ---------------------------------- |
| Decimal           | `1000000`      | Standard                           |
| Decimal separated | `1_000_000`    | Underscore separator, any position after first digit |
| Hexadecimal       | `0xFF`         | Case-insensitive (`0xFF` = `0XFF`) |
| Binary            | `0b1010_1100`  | Separators allowed                 |
| Octal             | `0o777`        | Explicit prefix — no bare `077`    |

```bestie
val mask: uint32  = 0xFF00_FF00
val flags: uint8  = 0b0000_1111
val perms: uint16 = 0o755
val million: int  = 1_000_000
```

### 8.2 Float Literals

| Form                | Example       |
| ------------------- | ------------- |
| Standard            | `3.14`        |
| Scientific notation | `1.5e10`      |
| Negative exponent   | `2.0e-4`      |
| Separated           | `1_234.567_8` |

```bestie
val pi:    float64 = 3.141_592_653
val small: float32 = 1.0e-9
```

### 8.3 Type Suffixes

Literal type is inferred from context. When explicit typing is needed, use `as`:

```bestie
val x = 255 as uint8
val y = 3.14 as float32
```

No implicit promotion. No suffix syntax — `as` keeps it consistent with the rest of casting.

---

## 9. Range Type

`range<T>` is a **core type** representing a contiguous sequence of values. Ranges are compile-time resolved when bounds are constants.

### 9.1 Syntax

| Operator | Meaning       | Example   |
| -------- | ------------- | --------- |
| `..`     | Exclusive end | `0..10`   → 0 to 9  |
| `..=`    | Inclusive end | `0..=10`  → 0 to 10 |

```bestie
val r1 = 0..10          // range<int>, exclusive: 0–9
val r2 = 0..=10         // range<int>, inclusive: 0–10
val r3 = 'a'..='z'      // range<char>
```

### 9.2 Compile-Time Resolution

When bounds are compile-time constants, the range is **fully resolved at compile time** — no runtime object created:

```bestie
for (i in 0..10) { ... }    // compile-time: unrolled or lowered to a counter
```

When bounds are runtime values, a lightweight `range<T>` value is created on the stack — no heap allocation.

### 9.3 Stored Ranges

Ranges are first-class values and can be stored:

```bestie
val r: range<int> = 0..100
val mid = r.mid()        // 50
val size = r.size()      // 100
val has = r.contains(42) // true
```

### 9.4 Rules

* `range<T>` requires `T impl Comparable`
* Ranges are value types — copied on assignment, no allocation
* Empty ranges are valid: `5..5` (exclusive) is empty
* Reversed ranges (`10..0`) are empty, not an error
* Ranges implement `Iterable<T>`

---

## 10. String Formatting

Bestie core provides string formatting and I/O as built-in functions — no import required.

### 10.1 String Interpolation

Embed expressions directly in string literals using `${}`:

```bestie
val name = "Alice"
val age  = 30
val msg  = "Hello ${name}, you are ${age} years old"
```

* Expression inside `${}` is evaluated and converted to `str`
* Compile-time constant expressions are resolved at compile time
* No hidden allocation for simple value types

### 10.2 `printf` — Formatted Output

Writes a formatted string to stdout:

```bestie
printf("Hello %s, age %d\n", name, age)
printf("Value: %.2f\n", 3.14159)
```

Format specifiers follow C conventions (`%s`, `%d`, `%f`, `%x`, etc.).
Types are checked at compile time when the format string is a literal.

### 10.3 `format` — Formatted String

Returns a formatted `str` without printing:

```bestie
val msg: str = format("User %s has %d items", name, count)
```

Same format specifiers as `printf`. Compile-time verified when format string is a literal.

### 10.4 `print` / `println` — Basic Output

```bestie
print("Hello")          // no newline
println("Hello")        // with newline
println(42)             // works on any type with str conversion
```

### 10.5 `input` / `inputf` — Input

```bestie
val line: str  = input()                  // reads a line from stdin
val name: str  = inputf("Enter name: ")   // prints prompt, then reads a line
```

`inputf` is a convenience wrapper — it prints the prompt and immediately reads the response. No format parsing on the input side; use the `!` error return if parsing is needed.

---

## 11. Program Entry Point

### 11.1 `main` Signature

`main` is the program entry point. It implicitly returns `void` — no return type annotation needed.

```bestie
fun main() {
    println("Hello, Bestie")
}
```

With command-line arguments:

```bestie
fun main(args: list<str>) {
    println(args[0])
}
```

Both forms are valid. `args` contains the raw command-line arguments as strings. Structured argument parsing lives in `std-api.cli`.

### 11.2 Rules

* `main` always returns `void` — writing the return type is unnecessary and not idiomatic
* `main` may declare `args: list<str>` as its only parameter — omitting it is fine
* `main` may return `! ErrorSet` for propagating startup errors:

```bestie
fun main(args: list<str>): void ! StartupError {
    val cfg = try loadConfig(args)
    run(cfg)
}
```

* `main` is the sole entry point — no `static void main` ceremony
* `main` executes on the process entry OS thread with `threadOs` semantics

---

## 12. Control Flow

### `if`

`if` is both a **statement** and an **expression**.

As an expression, `if` must produce a value:

```bestie
val x: int = if (cond) 4 else 0
```

If a value does not have a natural empty representation, `Option<T>` must be used.

As a statement, `if` may omit `else`.
When used inside a function without `else`, the function becomes **partial**:

```bestie
fun f(): bool ? = if (cond) return true
```

See `fp.md` for partial functions.

Properties:

* Statement and expression
* Expressions must return values
* Missing branch → partial function (`?`)

### `switch`

`switch` is both a **statement** and an **expression**.

Properties:

* No fallthrough
* Exhaustiveness enforced when expression
* Fully compile-time analyzable

```bestie
val x = switch (v) {
  1 => 10
  2 => 20
  else => 0
}
```

---

## 13. Loops

Supports:

* `for`
* `for in`
* `while`

Loops may include an `else` clause (similar to Python) when the loop completes without `break`.

Loops may be used as **expressions when compile-time resolvable**:

```bestie
val x: int = for (i = 0; i < 5; i++) i + 5
val xs: list<int> = for (i in 0..3) i * 2
```

While loop example:

```bestie
var i = 0
while (i < 10) {
  print(i)
  i += 1
}
```

This provides comprehension-style behavior without stream abstractions.

---

## 14. Syntax Rules

* Parentheses required for control flow
* Braces required for multi-line bodies
* Formatting enforced by tool, not compiler

---

## 15. Operators

Logical, bitwise, identity, and compile-time introspection operators supported.

No hidden operator behavior.

---

## 16. Functions (Overview)

* Static dispatch by default
* No hidden allocation
* Compile-time resolvable

See `fp.md`.

---

## 17. OOP (Overview)

* Closed classes by default
* Explicit inheritance
* Protocol-driven polymorphism

See `oop.md`.

---

## 18. Memory Model (Overview)

Manual, deterministic memory model:

* `own`
* `ref`
* `ptr<T>`
* Explicit unsafe boundaries for low-level operations (`ptr`, FFI, manual free)

See `memory.md`.

---

## 19. Concurrency (Overview)

* Explicit
* Compile-time validated
* No hidden sharing

See `concurrency.md`.

---

## 20. Annotations

Compile-time only, zero runtime cost.

See `annotations.md`.

---

## 21. Error Handling

Bestie avoids hidden null-like states and hidden exception flow.

Mechanisms:

* Complete returns
* Partial returns (`?`)
* `option<T>`
* Explicit errors

See `exceptions.md`.

---

## 22. Integer Overflow

Overflow behavior in Bestie is **defined, deterministic, and zero-cost in release builds**.

### 18.1 Unsigned Integers — Always Wrap

Unsigned overflow always wraps, unconditionally:

```bestie
val x: uint8 = 255
val y = x + 1   // y = 0, always
```

### 18.2 Signed Integers — Wrap in Release, Trap in Debug

In release builds, signed overflow wraps (two's complement). No cost, no surprise.
In debug builds, signed overflow traps at runtime to surface bugs early.

```bestie
val x: int8 = 127
val y = x + 1   // release: y = -128
                // debug:   runtime trap
```

Build mode is set in `bestie-project.toml`. No per-expression overhead in release.

### 18.3 Explicit Overflow Operators

When wrapping, saturating, or checked behavior is intentionally required, use explicit operators:

| Operator | Behavior                              | Use case                          |
| -------- | ------------------------------------- | --------------------------------- |
| `+%`     | Always wraps (unsigned semantics)     | Hash functions, ring buffers      |
| `-%`     | Always wraps                          | Modular arithmetic                |
| `*%`     | Always wraps                          | Fixed-width math                  |
| `+\|`    | Saturates at type max/min             | DSP, graphics, clamped counters   |
| `-\|`    | Saturates at type max/min             | Audio, signal processing          |
| `*\|`    | Saturates at type max/min             | Image pixel math                  |
| `+!`     | Always traps on overflow              | Safety-critical paths             |
| `-!`     | Always traps on overflow              | Checked arithmetic                |
| `*!`     | Always traps on overflow              | Checked arithmetic                |

```bestie
val a = x +% y    // wraps
val b = x +| y    // saturates
val c = x +! y    // traps
```

Rules:

* Explicit operators override build-mode behavior
* No hidden branching in release for `+%` and `+|`
* `+!` compiles to a checked instruction; cost is explicit and local

---

## 23. `defer`

`defer` schedules a statement to execute at the **end of the enclosing scope**, regardless of how the scope exits — normal return, early return, or exception unwind.

It is compile-time lowered. There is no runtime mechanism, no allocation, and no overhead.

### 19.1 Basic Usage

```bestie
fun readConfig(path: str): str ! IoError {
    val f = try file.open(path)
    defer f.close()              // runs on every exit path

    return try f.readAll()
}
```

### 19.2 Multiple Defers — LIFO Order

Multiple `defer` statements in the same scope run in **reverse declaration order**:

```bestie
val db = try db.connect()
defer db.close()          // runs second

val tx = try db.beginTx()
defer tx.rollback()       // runs first
```

### 19.3 Defer Inside Loops

`defer` inside a loop fires at the **end of each iteration**, not the end of the function:

```bestie
for (item in items) {
    val f = try file.open(item.path)
    defer f.close()       // closes after each iteration
    process(f)
}
```

### 19.4 Rules

* `defer` executes at the end of its **enclosing scope block**, not the function
* `defer` body cannot use `return`, `throw`, or `try`
* `defer` captures the variables it references **by binding** at declaration time
* Multiple defers in one scope execute **LIFO**
* Compile-time lowered — no runtime mechanism

---

## 24. Pattern Matching

Pattern matching in Bestie is an **extended `switch`** — exhaustive, compile-time verified, and zero-overhead.

Every `switch` must be exhaustive. Missing cases are a **compile-time error**.

### 24.1 Matching Primitives

```bestie
switch (status) {
    case 200 => println("OK")
    case 404 => println("Not Found")
    case 500 => println("Server Error")
    case _   => println("Other")
}
```

`_` is the wildcard — matches anything, must come last.

---

### 24.2 Matching Enums (with Payloads)

Enum variants with payloads are destructured inline:

```bestie
switch (result) {
    case result.Ok(val value) => println("Got: ${value}")
    case result.Err(val err)  => println("Error: ${err}")
}
```

Enum variants without payloads match directly:

```bestie
switch (opt) {
    case option.Present(val user) => greet(user)
    case option.Not_Present       => println("no user")
}
```

Exhaustiveness is enforced — all variants must be covered or a wildcard must be present.

---

### 24.3 Matching Data Classes (Destructuring)

```bestie
data class Point { x: int, y: int }

switch (pt) {
    case Point(x: 0, y: 0) => println("origin")
    case Point(val x, y: 0) => println("on x-axis at ${x}")
    case Point(x: 0, val y) => println("on y-axis at ${y}")
    case Point(val x, val y) => println("at ${x}, ${y}")
}
```

Fields not mentioned in a pattern are not bound and are ignored.

---

### 24.4 Matching Ranges

```bestie
switch (score) {
    case 90..=100 => println("A")
    case 80..=89  => println("B")
    case 70..=79  => println("C")
    case _        => println("F")
}
```

Range patterns use the same `..` / `..=` syntax as range literals.

---

### 24.5 Matching Tuples

```bestie
switch ((x, y)) {
    case (0, 0)       => println("origin")
    case (val a, 0)   => println("x-axis: ${a}")
    case (0, val b)   => println("y-axis: ${b}")
    case (val a, val b) => println("${a}, ${b}")
}
```

---

### 24.6 Guards

A `case` may include a guard condition with `if`. The guard is evaluated only when the pattern matches:

```bestie
switch (result) {
    case result.Ok(val n) if n > 0 => println("positive: ${n}")
    case result.Ok(val n)          => println("non-positive: ${n}")
    case result.Err(val e)         => println("error: ${e}")
}
```

Guards do not affect exhaustiveness analysis — the compiler treats guarded cases as partial.

---

### 24.7 Binding Captures

`val` inside a pattern binds the matched value as a local immutable:

```bestie
case option.Present(val user) => greet(user)
```

`var` binds a mutable copy:

```bestie
case option.Present(var user) => { user.name = "updated"; save(user) }
```

---

### 24.8 Rules

* Every `switch` must be exhaustive — missing cases are a **compile-time error**
* `_` wildcard satisfies exhaustiveness and must be the last case
* Pattern matching is **compile-time lowered** — no runtime type tags, no dispatch tables
* Guards do not affect exhaustiveness; a guarded case does not count as covering its pattern
* Matching is **structural** for `data class` and `enum` with payloads
* Nested patterns are supported

---

## 25. Conditional Compilation

Bestie uses `when` blocks for conditional compilation. `when` is evaluated entirely at **compile time** — the rejected branch is completely eliminated from the binary. There is no runtime branching.

```bestie
when (target.os == "linux") {
    val path = "/etc/config"
}
when (target.os == "windows") {
    val path = "C:\\config"
}
```

---

### 25.1 The `target` Object

`target` is a compile-time constant object with the following fields:

| Field           | Type     | Example values                          |
| --------------- | -------- | --------------------------------------- |
| `target.os`     | `str`    | `"linux"`, `"windows"`, `"macos"`, `"freebsd"` |
| `target.arch`   | `str`    | `"x86_64"`, `"arm64"`, `"riscv64"`     |
| `target.bits`   | `int`    | `32`, `64`                              |
| `target.endian` | `str`    | `"little"`, `"big"`                     |

All fields are resolved at compile time to string or integer constants.

---

### 25.2 Custom Flags

Custom flags are declared in the build configuration and accessed via `target.flag`:

```bestie
when (target.flag("ENABLE_SIMD")) {
    // SIMD path
}
```

Flags not declared are `false` by default — no runtime check, no linker symbol.

---

### 25.3 `when` as Expression

`when` can be used as an expression when both branches produce the same type:

```bestie
val pageSize: int = when (target.os == "windows") { 4096 } else { 8192 }
```

The unchosen branch is completely eliminated. The result is a compile-time constant when both sides are constant.

---

### 25.4 `when`/`else if`/`else`

```bestie
when (target.arch == "x86_64") {
    useAvx()
} else when (target.arch == "arm64") {
    useNeon()
} else {
    useGeneric()
}
```

---

### 25.5 Rules

* `when` conditions must be **compile-time evaluable** — runtime expressions are a compile-time error
* The rejected branch is **completely eliminated** — it is not type-checked as dead code
* `when` is not a runtime `if` — it does not produce branches in the binary
* `when` may appear at module level (top-level declarations), inside functions, and as expressions
* Nesting is allowed

---

## 26. Thread-Local Storage

`threadlocal` is a **storage modifier** — not a class, not a wrapper, not a container.

A `threadlocal` declaration gives each OS thread its **own independent copy** of the variable. Access is direct, with no `.get()`, `.set()`, or any other wrapper indirection.

```bestie
threadlocal val requestId: str = ""
threadlocal var callDepth: int = 0
```

---

### 26.1 Usage

Access and mutation are identical to any other variable:

```bestie
threadlocal var counter: int = 0

fun tick() {
    counter += 1
}
```

There is no ceremony. `counter` is a variable. The compiler routes reads and writes to the calling thread's copy.

---

### 26.2 Semantics

* Each `threadOs` thread gets its **own independent copy**, initialized from the declared initializer
* `threadLight` fibers share the copy of the OS thread they run on
* Initialization is **per-thread at first access** (lazy, but compile-time lowered to a TLS slot — no heap allocation)
* The initializer expression must be a **compile-time constant** or a pure function of compile-time constants
* Copies are independent — writes in one thread are invisible to others

---

### 26.3 Scope

`threadlocal` is allowed at:

* **Module level** (top-level declaration)
* **Function level** (static local — initialized once per thread, not once per call)

`threadlocal` inside a regular expression or block (non-static context) is a **compile-time error**.

---

### 26.4 Rules

* `threadlocal val` — immutable per-thread binding (the value is constant for that thread's lifetime)
* `threadlocal var` — mutable per-thread binding
* The initializer must be compile-time constant
* No sharing across threads — compiler enforces this for `own` values
* `ptr` into a `threadlocal` from another thread is legal but programmer-responsibility (same as all `ptr` use)
* No runtime overhead beyond the platform TLS mechanism (hardware register + offset on x86_64/arm64)
* No `@threadlocal` annotation, no `ThreadLocal<T>` class — `threadlocal` is a first-class storage modifier

---

## 27. Stability

* Core is sealed
* Backward compatibility is mandatory
* Experimental features require flags
* Higher layers evolve independently

# Bestie Core — Language Syntax

## 1. Overview

Bestie is a native, compiled programming language designed from first principles for **systems programming and backend engineering**.

Bestie is not built around a single paradigm.
Object-oriented programming, functional programming, and low-level control are treated as **explicit tools**, not ideologies.

Systems and backend engineers share **one language and one memory model** — there is no "safe subset" and no walled-off `unsafe` mode. Raw pointers, manual memory, and FFI are first-class citizens, expressed through explicit, searchable constructs (`ptr<T>`, `@trusted`, manual `free`) rather than a block that implies you've left the language. Both audiences are meant to feel at home in the same core.

Bestie prioritizes:

* Compile-time resolution
* Deterministic execution
* Predictable memory layout
* Zero-cost abstractions
* Long-term stability

### First program

```bestie
import bestie.api.io.println

fun main() {
    println("Hello, Bestie")
}
```

A first program is a function, not a class. Console output talks to the outside world, so it lives in `bestie.api.io` — not in core. Interpolation (`"Hello ${name}"`) is core syntax. See §10 and §11, and `std-api/io.md`.

---

## 2. Core Design Principles

The Bestie compiler resolves **everything that is resolvable** at compile time, including:

* Type inference
* Generic specialization (monomorphization)
* Memory layout, padding, and alignment
* Protocol dispatch
* Inline expansion
* Ownership accounting (`own` discharged exactly once)
* `defer` / destructor placement
* Builder chains
* Error handling paths
* Loop lowering and unrolling opportunities
* Devirtualization
* Concurrency safety for ownership/sharing rules (`own` cannot be implicitly shared; `ptr` sharing is programmer-owned)

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

### 3.1 Lexical Structure

This section defines how source text becomes tokens. Everything here is fixed for the life of the language.

#### 3.1.1 Source Files

* A source file is **UTF-8** encoded, with no byte-order mark. A BOM is a compile-time error, not silently skipped.
* The extension is `.bst` (`modules-and-packaging.md` §2).
* Line endings may be `LF` or `CRLF`; both are read as one line terminator. Emitted diagnostics always count lines the same way.

#### 3.1.2 Comments

| Form | Meaning |
| ---- | ------- |
| `// text` | Line comment — runs to the end of the line |
| `/* text */` | Block comment — **nests**, so `/* a /* b */ c */` is one complete comment |
| `/// text` | Documentation comment on the following declaration |
| `/** text */` | Block documentation comment on the following declaration |

```bestie
// a line comment

/// The number of retries before the connection is abandoned.
/// Applies per host, not per request.
const MAX_RETRIES: int = 3

/*  Block comments nest, so this
    /* including this inner one */
    is a single comment and code below stays commented out.  */
```

Block comments nest so that commenting out a region containing comments works. An unterminated block comment is a compile-time error.

Documentation comments attach to the **immediately following** declaration; one that precedes no declaration is a compile-time error. They are consumed by `bestie doc` and are not otherwise part of program semantics. Comments are never tokens: they cannot appear inside a string literal, and a `//` inside a string is ordinary text.

#### 3.1.3 Identifiers

An identifier starts with a Unicode letter or `_`, followed by any number of Unicode letters, digits, or `_`.

* Identifiers are **case-sensitive**.
* A bare `_` is not an identifier — it is the discard pattern (§4.6).
* Identifiers are compared by exact code points. Bestie does **not** apply Unicode normalization, so two identifiers that render identically but differ in code points are different identifiers; the compiler warns when a file contains such a pair.
* There is no length limit and no sigil or prefix convention carrying meaning.

#### 3.1.4 Reserved Words

Reserved in every position. None may be used as an identifier:

```
and         as          break       case        catch       class
const       continue    data        defer       deinit      else
enum        errors      ext         false       for         foreign
fun         if          impl        import      in          init
is          move        not         or          own         package
permits     private     protected   public      protocol    ref
return      sealed      super       switch      this        threadlocal
true        try         type        val         value       var
when        while       _
```

**Contextual keywords** have meaning only in one position and remain usable as identifiers elsewhere: `get` and `set` (property accessors, `oop.md` §10), `internal` (visibility), and `fn` (the optional function-type prefix, `fp.md` §6.1).

Built-in **type names** — `int`, `uint`, `float`, `bool`, `char`, `str`, `byte`, `array`, `slice`, `range`, `tuple`, `ptr`, `thread` — are ordinary identifiers bound by the prelude, not reserved words. Shadowing one is legal and warned about, in exactly the way shadowing any prelude name is (§4.8.3).

#### 3.1.5 Statement Termination

A statement ends at a **newline**. The semicolon `;` is an optional separator for writing more than one statement on a single line:

```bestie
val a = 1
val b = 2

if (ready) { start(); log("started") }    // ; separates, does not terminate
```

A trailing `;` at the end of a line is legal but not idiomatic, and `bestie fmt` removes it. A line ending with an open delimiter (`(`, `[`, `{`) or a binary operator continues onto the next line, so an expression may be wrapped freely.

#### 3.1.6 String Literals

A `str` literal is enclosed in double quotes and is UTF-8 (`types.md` §3).

| Escape | Meaning |
| ------ | ------- |
| `\n` `\r` `\t` `\0` | Newline, carriage return, tab, NUL |
| `\\` | Backslash |
| `\"` | Double quote |
| `\'` | Single quote |
| `\$` | A literal `$` — suppresses interpolation |
| `\u{...}` | Unicode scalar by hex code point, 1–6 digits (`\u{41}`, `\u{1F600}`) |

Any other `\x` sequence is a compile-time error — there are no silently-ignored escapes.

```bestie
val path = "C:\\config\\app.toml"
val money = "\$${amount}"           // literal $, then an interpolated value
```

**Raw strings** are delimited by `"""`. They span lines, process **no** escapes, and perform **no** interpolation — the text between the delimiters is exactly the value:

```bestie
val pattern = """\d+\.\d+"""        // backslashes are literal
val help = """
Usage: app [options]
  -v   verbose ${not interpolated}
"""
```

A raw string cannot contain `"""`. When the opening delimiter is followed immediately by a newline, that first newline is not part of the value; the closing delimiter's own line and its leading indentation are likewise excluded, and that indentation is stripped from every line.

#### 3.1.7 Character Literals

A `char` literal is enclosed in single quotes and holds exactly **one Unicode scalar** (`types.md` §2.6):

```bestie
val a = 'A'
val nl = '\n'
val emoji = '\u{1F600}'
```

The same escape table applies, except `\$` which is meaningless outside interpolation. A literal containing zero scalars, more than one scalar, or a surrogate code point is a compile-time error. `'ab'` is an error, not a truncation.

Numeric literal forms are in §8; interpolation is in §10.

---

## 4. Variables and Bindings

Bestie provides three binding forms.

### 4.1 `const` — Compile-Time Constant

`const` defines a binding whose value is resolved **entirely at compile time** — both the binding and the value are permanently immutable.

```bestie
const PI: float64 = 3.14159265358979323846
const MAX_CONNECTIONS: int = 1024
const APP_NAME: str = "bestie-server"
const BUFFER_SIZE: int = MAX_CONNECTIONS * 64   // other consts allowed
```

**Valid right-hand-side expressions:**

* Literals (`42`, `3.14`, `"hello"`, `true`)
* Other `const` values
* Arithmetic on `const` values — evaluated at compile time
* `@pure` function calls where all arguments are `const` — the call is evaluated at compile time and the result becomes the constant's value

**Valid scopes:**

| Scope | Behavior |
| ----- | -------- |
| Module level | Placed in `.rodata`. If never addressed (no `ptr` to it), inlined as an immediate operand everywhere — zero memory access. |
| Class / protocol body | Same as module level; scoped to the type namespace. |
| Function body (local const) | Always inlined as an immediate — no stack slot allocated, no load instruction emitted. |

**Invalid:**

* `const` whose RHS contains a runtime value (`val`, `var`, function return, I/O) — compile error
* `const` function parameter — parameters are always runtime values; use a type constraint instead

**Object file behavior:**

A `const` that is never taken by address compiles to **zero bytes of data** — the value is substituted as an immediate operand at every use site. When its address is taken, it occupies a single slot in `.rodata` shared across the entire binary.

```bestie
// function-level const: inline into every use, no stack frame impact
fun circleArea(r: float64): float64 {
    const TAU: float64 = 6.28318530717958647692
    return TAU * r * r / 2.0     // TAU is an immediate in the generated instruction
}
```

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

> **Two axes of `val` — binding vs. field.** The `val` keyword is used in two distinct positions, and they mean different things:
> * **`val` binding** (`val x = ...`) — the *binding* cannot be rebound. It says nothing about whether the storage it designates is writable; that depends on the type. Notably, taking `.address()` of a `val T` binding yields a **mutable** `ptr<T>`, because the binding designates writable storage (see `core/memory.md` §8.3 and §10.1.2).
> * **`val` field** (`val name: str` in a class / `data class` body) — the *field* itself cannot be mutated after construction. Field-level immutability is defined in `core/oop.md` (field declarations and accessors) and summarized per type in `core/immutability.md`.
>
> In short: a `val` binding means "this name cannot be rebound"; a `val` field means "this field cannot be mutated." The keyword is the same; the axis is different.

> For a complete per-type breakdown of what `val`, `@immutable val`, `.immutable`, and `const` each prevent, see `core/immutability.md`.

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

### 4.7 Definite Assignment

Bestie enforces **definite assignment** for all local bindings.
A `val` or `var` must be assigned before it is read. Reading an uninitialized local is a **compile-time error**.

```bestie
val x: int
print(x)       // ❌ compile-time error: 'x' used before initialization

val y: int = 5
print(y)       // ✅
```

The compiler tracks every reachable control-flow path. A binding that may be uninitialized on **any** path is rejected:

```bestie
val result: int

if (cond) {
    result = 42
}
print(result)   // ❌ compile-time error: 'result' may not be initialized on the false branch
```

Correct forms:

```bestie
val result: int = if (cond) 42 else 0   // ✅ always initialized

fun maybeAnswer(): int ? {
    if (cond) { return 42 }
    return                          // ✅ absence is explicit
}
val result = maybeAnswer()
```

Rules:

* No implicit zero-initialization of locals — the programmer is always in control
* Conditional initialization requires either a guaranteed `else` branch or `T ?`
* The same rule applies inside loops — a binding declared before a loop must be assigned before the loop can read it
* `val` bindings assigned exactly once satisfy definite assignment; assigning a second time is a compile-time error

---

### 4.8 Blocks, Scopes, and Shadowing

Bestie uses **lexical (static) scoping**. Every name is resolved at compile time against the textually enclosing scopes — never at runtime.

#### 4.8.1 Blocks

A block is a brace-delimited group of statements `{ ... }`. Every block introduces a **new nested scope**.

```bestie
fun example(): void {
    val a = 1
    {
        val b = a + 1     // inner block can see `a`
        print(b)
    }
    // `b` is not visible here — its scope ended with the inner block
}
```

Properties:

* A block scope begins at `{` and ends at the matching `}`
* Bindings declared in a block are destroyed (and `defer`s run, see section 23) when the block exits
* Inner scopes can read names from enclosing scopes; enclosing scopes cannot see names declared in inner scopes
* The bodies of `if`, `switch`, loops, and function literals are blocks and follow the same rules

#### 4.8.2 Scope Kinds

| Scope | Introduced by | Notes |
| ----- | ------------- | ----- |
| Module / file | top of a file | `var` is forbidden here; file-level `val` requires `@immutable` |
| Type body | `class` / `data class` / `protocol` / `enum` | members and `const`s live in the type namespace |
| Function | `fun` parameter list + body | parameters live in the function's top scope |
| Block | `{ ... }` | including `if`/`switch`/loop bodies and function-literal bodies |

Name resolution proceeds **innermost-out**: the compiler searches the current scope first, then each enclosing scope, stopping at the first match.

#### 4.8.3 Shadowing

A binding may **shadow** a binding of the same name from an **enclosing** scope. The inner name hides the outer one for the remainder of the inner scope; the outer binding is untouched and becomes visible again once the inner scope ends.

```bestie
val x = 10

fun f(): void {
    print(x)          // 10 — the outer `x`
    val x = 20        // shadows the outer `x` within this function body
    print(x)          // 20
    {
        val x = 30    // shadows again in this block
        print(x)      // 30
    }
    print(x)          // 20 — inner block ended
}
```

Rules:

* Shadowing is allowed **only across nested scopes**. The inner declaration must live in a strictly more nested scope than the binding it shadows.
* **Redeclaring a name in the same scope is a compile-time error** — there is no same-scope shadowing or rebinding via re-declaration. (Mutating an existing `var` with `=` is assignment, not shadowing.)
* A local binding may shadow a function parameter (the body block is nested relative to the parameter scope); two parameters may **not** share a name.
* Shadowing never changes the type or mutability of the outer binding — each binding is independent.
* Shadowing is purely lexical and resolved at compile time. It involves no dispatch and no runtime cost.

The compiler may emit a **lint warning** for shadowing that is likely unintentional, but shadowing across scopes is legal.

#### 4.8.4 Shadowing vs. Overriding

Shadowing and overriding are unrelated mechanisms and must not be confused:

| | Shadowing | Overriding |
| --- | --------- | ---------- |
| Applies to | local bindings / variables | `@virtual` methods in a class hierarchy |
| Mechanism | a name in an inner scope hides one in an outer scope | a subclass replaces a base method's implementation |
| Resolution | lexical, **compile time** | dynamic dispatch, **runtime** (vtable or sealed tag) |
| Requires | nothing — just a nested declaration | `@virtual` on the base, `@override` on the subclass |
| Cost | zero | one dynamic dispatch |

Crucially, Bestie has **no member hiding**. A subclass cannot silently redeclare a base field or non-`@virtual` method to shadow it:

* Redeclaring an inherited **field** is a compile-time error.
* Redeclaring a non-`@virtual` **method** is a compile-time error — there is no static "method hiding" as in some languages.
* The only legal way for a subclass to provide a new implementation of an inherited method is to **override** a `@virtual` method with `@override`.

This keeps method resolution unambiguous: a call either dispatches statically to a single definition or dynamically to a `@virtual` override. See `core/oop.md` §6 (Inheritance & Override Rules) for the full override semantics.

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
| `float`                           | Target-default | The platform's natural IEEE-754 float (`float64` on all current mainstream targets). Use when you want "a float" without committing to a width. |
| `bool`                            | 1-bit logical | `true` / `false`                         |
| `char`                            | 32-bit        | Unicode scalar value (valid codepoint)   |

`double` does not exist. Use `float64`.

**`int` and `uint` — pointer-sized by design:**

`int` and `uint` are not ambiguous — their width is the pointer width of the target platform. They exist specifically for indices, sizes, counts, and pointer arithmetic. Using `int32` or `int64` is correct when the bit width matters independently of the platform.

Mixing `int` with fixed-width types (`int32`, `int64`, etc.) requires an explicit cast. No silent narrowing or widening.

**`float` — target-default by design:**

Like `int` and `uint`, `float` is resolved at compile time to the platform's natural floating-point type — `float64` on every current mainstream target. It is the backend-friendly default for general-purpose math when you do not want to commit to a width; reach for `float32` or `float64` explicitly when the bit width matters (storage layout, SIMD, wire formats, GPU interop).

Mixing `float` with fixed-width float types (`float32`, `float64`) requires an explicit cast. No silent narrowing or widening.

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

`s[i]` returns `byte` (not `char`) because UTF-8 is variable-width: byte indexing is O(1), whereas codepoint indexing would be O(n) on a UTF-8 buffer. Bestie keeps `[]` as the O(1) byte path rather than hiding a decode behind it. See `core/types.md` §3 for the full rationale.

No hidden cost. The fast path stays fast. The correct path is explicit.

Properties:

* Binding immutability defaults to `val`/`const`
* Value mutability depends on type qualifiers
* Efficient copy semantics
* No hidden allocation

---

### 5.3 Core Collections

Bestie includes exactly one collection in the core language: `array<T>`.

`array<T>` is built-in and available without `import`.
It provides fixed-capacity contiguous storage, deterministic layout, and index-based access.

`{1, 2, 3}` is an **array**, not a Python/Java `List`. A growable sequence is `list<T>` in `bestie.lib.collections` — write `val xs: list<int> = {1, 2, 3}`. That annotation is how a newcomer opts into growth; the default stays the cheap form.

Core `array<T>` supports:

* `array<T>[n]` — sized declaration, capacity `n`, initially empty
* `array<T>[] = {v1, v2, ...}` — capacity and size inferred from a literal
* Indexing via `arr[i]` — panics on out-of-bounds access
* Direct index assignment via `arr[i] = value` — panics on out-of-bounds access
* Core methods: `add`, `get`, `size`, `capacity`, `isEmpty`, `isFull`
* Capacity is fixed at construction — adding past capacity is a panic

All dynamic and higher-level collections live in `bestie.lib.collections`, including:

* `list<T>` — dynamic, resizable, with `linked` and future sequence variations
* `set<T>`
* `map<K,V>`
* `deque<T>`
* `heap<T>`

Example:

```bestie
val arr : array<int>[5]
arr.add(1)
arr.add(2)

// Direct bracket syntax works for both reads and writes —
// equivalent to calling arr.get(i) and arr.set(i, value)
arr[0] = 10          // assign directly by index
val x = arr[2]       // read directly by index
val first = arr.get(0)   // method form also valid
```

---

### 5.4 Literal Type Inference and Disambiguation

Bestie uses the **binding's declared type** to resolve what a `{...}` or `{k: v, ...}` literal means. When no type is declared, the compiler applies a **default inference rule**.

#### Default inference rules (no type annotation)

| Literal form | Inferred type | Example |
| ------------ | ------------- | ------- |
| `{v, v, ...}` | `array<T>` | `val s = {1, 2, 3}` → `array<int>` |
| `{k: v, k: v, ...}` | the default `map<K,V>` | `val m = {1: "a", 2: "b"}` → `map<int,str>` |

`{1, 2, 3}` without a type annotation is always an `array<int>`, never a `list` or `set`. This is intentional: the default is the most efficient, lowest-overhead form.

A key-value literal infers `map<K,V>` in its **default representation**. Core fixes that the type is `map<K,V>` and that `K` must satisfy the cited `hash()` (§27); *which* representation is the default, and what other representations exist, is `bestie.lib.collections`' decision — see `std-lib/collections.md` §3.1. Core does not name the variation menu, so lib can extend it without a language change.

#### Explicit disambiguation

When the desired type differs from the default, an explicit type annotation on the binding overrides inference:

```bestie
val a = {1, 2, 3}                       // array<int>   — default
val b : list<int>    = {1, 2, 3}        // list<int>    — explicit
val c : set<int>     = {1, 2, 3}        // set<int>     — explicit (deduplicates)
val d : deque<int>   = {1, 2, 3}        // deque<int>   — explicit

val e = {1: "a", 2: "b"}               // map<int,str> — default representation
val f : map<int,str>.linked = {1: "a"} // explicit variation (see std-lib/collections.md)
```

The annotation drives the entire resolution — no ambiguity, no implicit coercion between types.

#### Compiler hints for partial annotation

When only a variation needs to change (not the full type), the annotation can target just the variation:

```bestie
val g : map<str,int>.tree = {"x": 1, "y": 2}   // tree-backed map from literal
val h : list<int>.linked  = {1, 2, 3}           // linked list from literal
```

#### Rules

* Without a type annotation, `{v, ...}` **always** resolves to `array<T>` — never `list`, `set`, or `deque`
* Without a type annotation, `{k: v, ...}` **always** resolves to `map<K,V>` in its default representation
* The compiler never silently picks a different collection type from a literal — it either uses the annotation or the default
* A mismatched annotation is a compile-time error: `val x : set<str> = {"a": 1}` is rejected because a map literal cannot satisfy a `set<str>` type
* Multi-dimensional array literals follow the same rule: `{{1,2},{3,4}}` without annotation infers `array<array<int>>`

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

### 6.2 Value Range Constraints (`in`)

A type may carry a **compile-time value range** using the `in` keyword. The range becomes part of the type's invariant — the compiler treats it as a guaranteed fact at every use site.

```bestie
type Score       as int32   in 0..=100
type Port        as uint16  in 1..=65535
type Probability as float64 in 0.0..=1.0
type AsciiChar   as uint8   in 0..=127
```

Range constraints work on **any numeric type** — integers, unsigned integers, and floats.

---

#### Construction

A constrained value must be constructed explicitly. The compiler validates at the construction point — never at use sites.

**From a compile-time constant** — validated at compile time, zero runtime cost:

```bestie
const MAX: Score = 100 as Score   // ✅ 100 is in 0..=100 — compile-time verified
const BAD: Score = 200 as Score   // ❌ compile error: 200 out of range 0..=100
```

**From a runtime value** — validated at runtime, returns an error on failure:

```bestie
val s = try (userInput as Score)   // ✅ returns ! RangeError if out of range
```

The check is a single compare + conditional branch at the construction site. After that, the value is trusted everywhere.

**Unchecked construction** — for performance-critical paths where the programmer can guarantee the value is in range:

```bestie
@trusted val s = (userInput as Score)   // no runtime check — programmer's responsibility
```

`@trusted` suppresses the check. It is visible in code review and searchable. It carries the same responsibility as `ptr<T>` — correctness is on the programmer. Out-of-range values in a `@trusted` cast produce undefined behavior.

---

#### Inline Constraints on Parameters

Range constraints can be applied inline on function parameters without declaring a named type:

```bestie
fun setVolume(level: int32 in 0..=100): void { ... }
fun lerp(t: float64 in 0.0..=1.0, a: float64, b: float64): float64 { ... }
```

An inline constraint is an anonymous constrained type. The same construction and validation rules apply at the call site.

---

After construction, the range is a fact the compiler uses: one check at the construction site, then no repeated checks, possible bounds-check elimination, niche tags, and overflow proofs. That is a compiler obligation, not something you write. See `docs/compiler/compiler-architecture.md` (layout compaction).

#### Rules

* The `in` range uses the same `..` (exclusive) / `..=` (inclusive) syntax as range literals
* Both bounds must be **compile-time constants**
* The range must be a valid subrange of the underlying type's full range — `uint8 in -1..=100` is a compile error
* `const` construction is always compile-time validated — no runtime check, no code emitted
* Runtime construction via `try (x as T)` emits one check at the construction site only
* Range information propagates through `type X as Y in range` — the newtype inherits the constraint
* Two constrained types with different ranges are distinct types even if they share the underlying type

---

### 6.3 Compact Representation Guarantee

The compiler is **obligated** to use the minimum valid representation for every type — enum tags, niches, sealed-class tags, `bool`, `char`, and packed class fields. You do not opt in. There is no `@layout` / `@stable` escape hatch in core; FFI that must match a C header uses `bestie.api.foreign` (`@repr(C)`).

How it packs, which niches exist, and what that does to the object file: `docs/compiler/compiler-architecture.md` (layout compaction).

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

## 10. String Interpolation

Embed expressions in string literals with `${}`:

```bestie
val name = "Alice"
val age  = 30
val msg  = "Hello ${name}, you are ${age} years old"
```

* The expression is evaluated and converted to `str`
* Compile-time constant expressions are resolved at compile time
* No hidden allocation for simple value types

Hosted console I/O (`print`, `println`, `input`, `printf`) lives in `bestie.api.io`. Structured data codecs live in `bestie.lib.format`.

---

## 11. Program Entry Point

### 11.1 `main` Signature

`main` is the program entry point. It implicitly returns `void` — no return type annotation needed.

```bestie
import bestie.api.io.println

fun main() {
    println("Hello, Bestie")
}
```

With command-line arguments (fixed argv — not a resizable `list`):

```bestie
fun main(args: array<str>) {
    println(args[0])
}
```

Both forms are valid. Structured argument parsing lives in `bestie.api.cli`.

### 11.2 Rules

* `main` always returns `void` — writing the return type is unnecessary and not idiomatic
* `main` may declare `args: array<str>` as its only parameter — omitting it is the beginner form
* `main` may be fallible (`! ErrorSet`) for startup errors:

```bestie
fun main(args: array<str>): void ! StartupError {
    val cfg = try loadConfig(args)
    run(cfg)
}
```

* `main` is the sole entry point — no `static void main` ceremony
* `main` executes on the process entry OS thread with `thread` semantics

---

## 12. Control Flow

### `if`

`if` is both a **statement** and an **expression**.

As a **statement**, `else` is optional. A missing `else` does not change the function's type. A function that can miss a `return` on some path is a compile-time error (incomplete), not silently `T ?`.

```bestie
fun f(cond: bool): int {
    if (cond) {
        return 1
    }
    return 0
}
```

As an **expression**, every branch must yield a value of the same type:

```bestie
val x: int = if (cond) 4 else 0
```

If you need “maybe no value”, use a `T ?` function or `if-let` — not a missing `else` that secretly changes the function type. See `fp.md` for partial functions.

Properties:

* Statement: `else` optional; missing `return` paths are errors
* Expression: all branches produce a value
* No implicit “this function is now partial” from a bare `if`

### `switch`

`switch` is both a **statement** and an **expression**.

Properties:

* No fallthrough
* **Always exhaustive** (statement or expression) — missing cases are a compile-time error; `_` covers the rest
* Fully compile-time analyzable

```bestie
val x = switch (v) {
  case 1 => 10
  case 2 => 20
  case _ => 0
}
```

`switch` decides at **runtime**. When the value being matched is a compile-time constant (e.g. `target.os`), prefer `when` (§25) — it resolves the choice at compile time and eliminates the unused branches instead of emitting them. The compiler warns when a `switch` branches only on compile-time constants.

---

## 13. Loops

Bestie has three loop forms: `for`, `for in`, and `while`. All three are **statements** — a loop is never an expression and never produces a value.

```bestie
for (i = 0; i < 10; i += 1) {
    work(i)
}

for (item in items) {
    process(item)
}

var i = 0
while (i < 10) {
    work(i)
    i += 1
}
```

#### What `for/in` requires

`for (x in xs)` is core syntax, so core defines what `xs` must provide. The loop compiles when `xs` satisfies **one** of the following, checked in this order:

1. **A compiler-known core sequence** — `array<T>`, `slice<T>`, `range<T>`, or a `str` iteration method (`chars()`, `bytes()`). The loop lowers to a direct index or pointer walk with no call. This is the zero-cost path and needs no protocol.
2. **A type exposing `iterator()`** — any type with a method

   ```bestie
   fun iterator(): I
   ```

   where `I` has

   ```bestie
   fun next(): T ?
   ```

   The loop desugars to:

   ```bestie
   // for (x in xs) { body }   ⇒
   val it = xs.iterator()
   while (true) {
       val x = it.next() else { break }
       body
   }
   ```

   `next()` returning absent (`T ?`, §21) ends the loop, via the `else` unwrap of `fp.md` §3.3. Because the desugaring is a static call on a concrete type, it is monomorphized and inlined like any other method — there is no dynamic dispatch and no allocation. This is the specified lowering, not a form you write by hand.

Anything else is a compile-time error:

```
error: 'Foo' is not iterable — 'for/in' requires a 'fun iterator()' whose result has 'fun next(): T ?'
```

**Where the protocols live.** The named protocols `Iterable<T>` and `Iterator<T>` are declared in `bestie.lib.patterns`, and every std-lib collection implements them. Core does not need that import to compile a `for` loop — it requires the *shape* above, and a lib protocol is one way to have it. But because core names `iterator()` and `next()` here, **those two method names are frozen**: they are part of the language contract and cannot be renamed or removed while the loop keyword exists (see §27). Everything else about `Iterable` / `Iterator` remains lib's to evolve.

This is the general rule for core: core defines what its syntax *means*, lib supplies types that satisfy it.

### 13.1 `break` — Leave the Loop

`break` ends the innermost enclosing loop immediately. Execution resumes at the first statement after that loop.

```bestie
for (item in items) {
    if (item.isPoisoned()) {
        break            // stop scanning entirely
    }
    process(item)
}
```

### 13.2 `continue` — Next Iteration

`continue` skips the rest of the current iteration and advances the innermost enclosing loop.

| Loop form | `continue` jumps to |
| --------- | ------------------- |
| `for (init; cond; update)` | the **update** clause, then the condition |
| `for (x in xs)` | the next element |
| `while (cond)` | the condition |

In a C-style `for`, the update clause **always** runs — `continue` cannot skip `i += 1` and cannot produce an accidental infinite loop.

```bestie
for (i = 0; i < 10; i += 1) {
    if (i % 2 == 0) {
        continue         // i += 1 still runs
    }
    work(i)
}
```

### 13.3 Labeled Loops

A loop may carry a **label** so that `break` / `continue` can target an enclosing loop instead of the innermost one.

```bestie
outer: for (row in rows) {
    for (cell in row) {
        if (cell.isEmpty()) {
            continue outer     // next row
        }
        if (cell.isFatal()) {
            break outer        // leave both loops
        }
        process(cell)
    }
}
```

Rules:

* A label is written `name:` immediately before `for`, `for in`, or `while`
* Labels attach **only to loops**. Labeling a bare block, an `if`, or a `switch` is a compile-time error — Bestie has no `goto` and no forward jump
* Label names live in their own namespace; a label never collides with a binding, type, or function of the same name
* `break name` / `continue name` must appear lexically inside the loop labeled `name`
* A label may not shadow an enclosing label of the same name
* An unused label is a compile-time error

The alternative to labels is a flag variable or an extracted function. Both read worse and neither is cheaper: `break outer` lowers to the same single jump.

### 13.4 `break` Is Not a `switch` Terminator

Bestie's `switch` has **no fallthrough** (§12), so `break` is never needed to end a case. Inside a loop, a `break` written in a `switch` case therefore targets the **loop**:

```bestie
for (msg in queue) {
    switch (msg.kind) {
        case Kind.Data  => handle(msg)
        case Kind.Close => break      // exits the for loop, not the switch
    }
}
```

This differs deliberately from C, Java, and C#, where the same code would only leave the `switch`. `break` has exactly one meaning in Bestie: leave a loop.

### 13.5 Interaction with `defer` and Ownership

`break` and `continue` are ordinary scope exits. Everything that happens on a normal exit still happens:

* Pending `defer` statements execute in LIFO order (§23) for every scope being left — `continue` runs the current iteration's defers; `break` runs the defers of every block it exits, innermost first
* Ownership accounting (`memory.md` §7) treats the `break` / `continue` path exactly like a `return` path: an `own` value that is still live when control leaves its scope must already have been discharged, or the compiler reports a leak

```bestie
for (path in paths) {
    val f = try file.open(path)
    defer f.close()          // runs on the continue path too
    if (f.isEmpty()) {
        continue
    }
    process(f)
}
```

### 13.6 Rules

* `break` and `continue` are **statements**, never expressions — they carry no value, and no loop yields one
* `break` / `continue` outside any loop is a compile-time error
* The unlabeled forms bind to the **innermost enclosing loop**
* Neither crosses a function boundary: a local `fun` or a lambda body declared inside a loop cannot `break` or `continue` that loop
* Neither may appear in a `defer` body (§23.4)
* Statements following `break` / `continue` in the same block are unreachable and are a **compile-time error**
* Both lower to a single direct jump — no runtime cost, no allocation

---

## 14. Syntax Rules

* Parentheses required for control flow
* Braces required for multi-line bodies
* Formatting enforced by tool, not compiler

---

## 15. Operators

Logical operators are words. Bitwise operators are symbols. One spelling each — the same choice as `ptr` / `.address()` over `&` / `*`.

Core owns the operator **inventory**, **precedence**, and **lowering**. Which types opt in to an overloadable operator is a library concern (§15.3).

### 15.1 Inventory

| Group | Operators | Notes |
| ----- | --------- | ----- |
| Arithmetic | `+` `-` `*` `/` `%` | `%` on integers only |
| Unary | `-x` `~x` `not x` | Negate, bitwise complement, boolean not |
| Overflow-explicit | `+%` `-%` `*%` · `+\|` `-\|` `*\|` · `+!` `-!` `*!` | Wrap / saturate / trap — §22.3 |
| Comparison | `<` `<=` `>` `>=` | Total order required |
| Equality | `==` `!=` | §15.4 |
| Logical | `and` `or` `not` | Words, not symbols. Short-circuiting |
| Bitwise | `&` `\|` `^` `~` `<<` `>>` | Integers only; **not** overloadable |
| Assignment | `=` · `+=` `-=` `*=` `/=` `%=` `&=` `\|=` `^=` `<<=` `>>=` | Statement, never an expression |
| Range | `..` `..=` | §9 |
| Access | `.` `[]` `()` `::` | Member, index, call, method reference |
| Type | `as` `is` | Cast (§6), runtime type test |
| Error / absence | `try` `catch` `else` | §21, `exceptions.md` |

`not` is boolean negation. `!` is **not** boolean not: it is the error-union operator (`T ! E`) and the overflow-trap suffix (`+!`). `&&` and `||` are not in the language.

```bestie
if (ready and not failed) { ... }
```

**Assignment is a statement.** `a = b` yields no value, so `if (x = 5)` is a compile-time error rather than a classic C bug. Compound assignment follows the same rule.

### 15.2 Precedence and Associativity

Highest binding first. Operators in the same row have equal precedence.

| # | Operators | Associativity |
| - | --------- | ------------- |
| 1 | `.` `[]` `()` `::` | left |
| 2 | `as` | left |
| 3 | `-x` `~x` `not x` · `try` | right (prefix) |
| 4 | `*` `/` `%` `*%` `*\|` `*!` | left |
| 5 | `+` `-` `+%` `-%` `+\|` `-\|` `+!` `-!` | left |
| 6 | `<<` `>>` | left |
| 7 | `&` | left |
| 8 | `^` | left |
| 9 | `\|` | left |
| 10 | `..` `..=` | none (non-associative) |
| 11 | `<` `<=` `>` `>=` · `is` | none (non-associative) |
| 12 | `==` `!=` | none (non-associative) |
| 13 | `and` | left |
| 14 | `or` | left |
| 15 | `catch` `else` | left |
| 16 | `=` `+=` `-=` … | right (statement) |

Notes:

* **Comparison and equality are non-associative.** `a < b < c` is a compile-time error, not a silently wrong `(a < b) < c`. The same applies to `a == b == c` and `0..5..10`.
* **`as` binds tighter than arithmetic**, so `x + 10.0 as Meters` is `x + (10.0 as Meters)`.
* **`try` binds to the following postfix expression**, so `try f(x) + 1` is `(try f(x)) + 1`.
* **Bitwise operators bind tighter than comparison** — unlike C, where `a & b == c` means `a & (b == c)`. In Bestie it means `(a & b) == c`, which is what the code looks like it says.
* Parentheses always override. The formatter (`bestie fmt`) does not insert or remove them.

### 15.3 Operator Lowering

An operator on a user-defined type lowers to a **method call**, resolved statically at compile time. There is no dynamic dispatch, no vtable, and no boxing — the call is monomorphized and inlined like any other.

| Expression | Lowers to |
| ---------- | --------- |
| `a + b` | `a.add(b)` |
| `a - b` | `a.sub(b)` |
| `a * b` | `a.mul(b)` |
| `a / b` | `a.div(b)` |
| `a % b` | `a.mod(b)` |
| `-a` | `a.neg()` |
| `a += b` | `a.addAssign(b)` |
| `a -= b` | `a.subAssign(b)` |
| `a *= b` | `a.mulAssign(b)` |
| `a /= b` | `a.divAssign(b)` |
| `a %= b` | `a.modAssign(b)` |
| `a[i]` | `a.get(i)` |
| `a[i] = v` | `a.set(i, v)` |
| `a == b` | `a.equal(b)` |
| `a != b` | `not a.equal(b)` |
| `a < b` | `a.compareTo(b) < 0` |
| `a <= b` | `a.compareTo(b) <= 0` |
| `a > b` | `a.compareTo(b) > 0` |
| `a >= b` | `a.compareTo(b) >= 0` |

Rules:

* **Primitives never lower.** On `int`, `float64`, `char`, `bool`, `ptr<T>`, and `str`, every operator in §15.1 is emitted directly as machine instructions. The table applies only to user-defined types.
* **Bitwise and logical operators are not overloadable.** The bitwise set is defined on machine integers; `and` / `or` / `not` are boolean control flow whose short-circuiting a method call cannot express. There is no lowering row for them and no protocol to implement.
* **Overflow-explicit operators are not overloadable.** `+%` / `+\|` / `+!` describe machine-integer behavior; on a user-defined type they are a compile-time error.
* Using an operator on a type that provides no matching method is a compile-time error, never a fallback.
* Newtypes (`type X as Y`, §6.1) inherit `Y`'s operators, including whatever `Y` lowers to.

**Where the protocols live.** The protocols that declare these methods — `Addable`, `Subtractable`, `Multipliable`, `Divisible`, `Modulable`, `Negatable`, `Indexable`, `IndexAssignable`, `Equable`, `Comparable` — are in `bestie.lib.utilities`, and a type opts in by implementing them. Core defines only the mapping above. Because core names these methods, **`add`, `sub`, `mul`, `div`, `mod`, `neg`, the `*Assign` forms, `get`, `set`, `equal`, and `compareTo` are frozen** (§27); the protocols themselves stay lib's to extend.

### 15.4 Equality

`==` compares **values**. What that means depends on the kind of type, and core fixes it:

| Kind | `a == b` means |
| ---- | -------------- |
| Primitives (`int`, `float64`, `char`, `bool`, …) | Numeric / bit equality. `float64.NAN == float64.NAN` is `false` — use `.isNaN()` |
| `str` | Byte-for-byte content equality |
| `tuple` | Field-by-field, in order |
| `value class`, `data class` | **Structural** — compiler-generated, field by field, recursively |
| `enum` | Same variant, and payloads equal structurally |
| `class`, `open class`, `abstract class` | **Identity** — the same object, i.e. the same address |
| `ptr<T>` | Address equality; the pointee is never read (`memory.md` §8.7) |
| Any type with `impl Equable` | `a.equal(b)` — the manual implementation wins |

This is what backs the claim in `oop.md` §3.1 that a `data class` has structural equality: the compiler derives it, and no import is required.

```bestie
data class Point { x: int, y: int }
class Node { val id: int }

Point.new(x: 1, y: 2) == Point.new(x: 1, y: 2)   // true  — structural
Node.new(id: 1)       == Node.new(id: 1)         // false — two distinct objects
```

There is **no separate identity operator** (`===`). The kind of the type already decides which comparison you get, and a `class` is exactly the kind that has identity. To compare two `class` values by content, implement `Equable` and say so in the type.

A derived structural `==` requires every field to be comparable; a field whose type is a `class` compares by identity, and that propagates outward. Deriving `==` on a type holding a `ptr<T>` compares addresses.

### 15.5 Introspection and Identity

* `is` — runtime type test (instanceof-equivalent): `x is T` yields `bool`. Meaningful for `\|virtual` and sealed hierarchies; statically resolved (and warned as redundant) when the type is known at compile time. See `core/oop.md` §5.5.
* `typeOf(x)` — compile-time type query
* `sizeOf(T)` — compile-time size in bytes
* `alignOf(T)` — compile-time alignment requirement in bytes
* `offsetOf(T, field)` — compile-time byte offset of a field within `T`'s packed layout (`memory.md` §18.1)

All five are resolved by the compiler and emit no code. `sizeOf`, `alignOf`, and `offsetOf` are valid in `const` initializers and `when` conditions (§25.2).

`offsetOf` reports the **compiler-chosen** offset, not the source declaration index — fields are reordered and packed (`memory.md` §18.1). It is the supported way to compute a field address without hand-arithmetic, and it is what allocators, MMIO regions, and `@repr(C)` interop need.

No hidden operator behavior.

---

## 16. Functions (Overview)

* Static dispatch by default
* No hidden allocation
* Compile-time resolvable
* Local (nested) functions for scoped helpers — no implicit capture
* Lambdas may capture only with explicit `[...]` lists (no environment closures)

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

* On a **stored slot** (field or collection element): `own` = we free, `ref` = we do not
* On a **call**: copy a value (`T`), move (`own T`), or point (`ptr<T>`). `class` is not copyable
* `slice<T>` is the one fat view that cannot be stored and cannot outlive its source
* First-class explicit low-level operations (`ptr`, FFI, manual free) — no `unsafe` block, just visible intent

See `memory.md`.

---

## 19. Concurrency (Overview)

* Explicit
* Compile-time validated
* No hidden sharing
* Core primitive: `thread` (1:1 OS thread)
* Fibers and coordination: `bestie.lib.concurrency`

See `concurrency.md`.

---

## 20. Annotations

Compile-time only, zero runtime cost.

See `annotations.md`.

---

## 21. Error Handling

Bestie avoids hidden null-like states and hidden exception flow.

Mechanisms:

* `T ?` — absence (core syntax)
* `T ! E` — recoverable failure (core syntax)

Named `option<T>` / `result<T, E>` are std-lib (`bestie.lib.utilities`) — same representation, not a second system, not core.

See `exceptions.md`, `types.md` §8.3–8.4, and `std-lib/util.md`.

---

## 22. Integer Overflow

Overflow behavior in Bestie is **defined, deterministic, and zero-cost in release builds**.

### 22.1 Unsigned Integers — Always Wrap

Unsigned overflow always wraps, unconditionally:

```bestie
val x: uint8 = 255
val y = x + 1   // y = 0, always
```

### 22.2 Signed Integers — Wrap in Release, Trap in Debug

In release builds, signed overflow wraps (two's complement). No cost, no surprise.
In debug builds, signed overflow traps at runtime to surface bugs early.

```bestie
val x: int8 = 127
val y = x + 1   // release: y = -128
                // debug:   runtime trap
```

Build mode is one of `debug`, `release`, or `safe` (`exceptions.md` §4.2). Core defines what each mode means; the toolchain decides which one is active for a given build — see `std-tools` and `modules-and-packaging.md` §3.2 for how a project selects it. No per-expression overhead in release.

### 22.3 Explicit Overflow Operators

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

`defer` schedules a statement to execute at the **end of the enclosing scope**, regardless of how the scope exits — normal return, early return, or error propagation through `try`.

It is compile-time lowered. There is no runtime mechanism, no allocation, and no overhead.

### 23.1 Basic Usage

```bestie
fun readConfig(path: str): str ! IoError {
    val f = try file.open(path)
    defer f.close()              // runs on every exit path

    return try f.readAll()
}
```

### 23.2 Multiple Defers — LIFO Order

Multiple `defer` statements in the same scope run in **reverse declaration order**:

```bestie
val db = try db.connect()
defer db.close()          // runs second

val tx = try db.beginTx()
defer tx.rollback()       // runs first
```

### 23.3 Defer Inside Loops

`defer` inside a loop fires at the **end of each iteration**, not the end of the function:

```bestie
for (item in items) {
    val f = try file.open(item.path)
    defer f.close()       // closes after each iteration
    process(f)
}
```

This includes iterations cut short by `continue`, and the final iteration when a `break` leaves the loop (§13.5).

### 23.4 Rules

* `defer` executes at the end of its **enclosing scope block**, not the function
* `defer` body cannot use `return`, `try`, `break`, or `continue` — a deferred statement runs *during* a scope exit and may not start another one
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

Named `option` / `result` constructors live in `bestie.lib.utilities` (`import` required). Signatures should still use `T ?` and `T ! E`.

```bestie
import bestie.lib.utilities.result

switch (result) {
    case result.Ok(val value) => println("Got: ${value}")
    case result.Err(val err)  => println("Error: ${err}")
}
```

Enum variants without payloads match directly:

```bestie
import bestie.lib.utilities.option

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
* Pattern matching is **compile-time lowered** to comparisons, branches, and sealed-tag tests where needed — no reflection and no runtime dispatch tables
* Guards do not affect exhaustiveness; a guarded case does not count as covering its pattern
* Matching is **structural** for `data class` and `enum` with payloads
* Nested patterns are supported

---

## 25. Compile-Time Conditionals (`when`)

`when` is Bestie's **compile-time conditional** — a branch resolved entirely by the compiler. The condition must be a compile-time-evaluable expression; the losing branch is **erased before code generation** and never appears in the binary. There is no runtime branching.

```bestie
when (target.os == "linux") {
    val path = "/etc/config"
} else when (target.os == "windows") {
    val path = "C:\\config"
}
```

Conditional compilation (selecting code per platform or build) is the most familiar use, but `when` is **not limited to it** — it is the general "decide this at compile time" construct (see §25.3 for the full range of uses).

### 25.1 Why Bestie Has Both `switch` and `when`

Bestie treats **systems programming as its first audience**, and systems code lives or dies by what it can resolve *before* the program runs. So Bestie promotes the compile-time branch to a first-class control construct alongside the runtime one. They are not two ways to do the same thing — they operate at different times and obey different rules:

* **`switch` (and `if`) — runtime, data-directed.** Branch on a value the program computes while running. Every arm is compiled, type-checked, and present in the binary. This is ordinary program logic.
* **`when` — compile-time, configuration-directed.** Branch on facts fixed before execution (target, build mode, flags, `const` values, `sizeOf`/`typeOf`). The losing arm is erased — not compiled, not type-checked, not in the binary.

The decisive difference — and the reason one keyword cannot cover both — is that **a `when` branch may contain code that does not even compile on the other target** (a Windows-only call inside a Linux build, an intrinsic that exists only on `arm64`). A runtime `switch` can never do this: it must compile every arm. Merging the two into one keyword would force an implicit rule ("if the condition happens to be constant, silently switch to erase-and-don't-type-check semantics"), which is exactly the kind of hidden behavior Bestie rejects. Two names, two clearly separated semantics.

> **Note for Java and Kotlin users.** The keywords do not map the way you expect:
> * **Java** has `switch` (runtime) and no `when` — Bestie's `switch` behaves as you'd expect; `when` is new.
> * **Kotlin** uses `when` as its *runtime* multi-way branch. **Bestie's `when` is the opposite — it is compile-time.** Bestie's *runtime* multi-way branch is `switch` (§12, §24). Kotlin does conditional compilation through build tooling, not a language keyword, so it is not a precedent for a compile-time `when`.
>
> In Bestie: **`switch`/`if` = runtime, `when` = compile-time.** Always.

---

### 25.2 Compile-Time Conditions

`when` conditions branch on any **compile-time-evaluable expression**, including:

* the `target` object — `target.os`, `target.arch`, `target.bits`, `target.endian`
* the `build` object — `build.mode`, `build.debug`
* custom build flags — `target.flag("NAME")`
* any `const` value or `@pure` call over constants (§2)
* compile-time type queries — `sizeOf(T)`, `typeOf(x)` (§15)

```bestie
when (target.flag("ENABLE_SIMD")) {
    // SIMD path — compiled only when the flag is set
}

when (build.debug) {
    logInvariants()      // present only in debug builds
}
```

A runtime expression as the condition is a **compile-time error**. See `core/constants.md` for the authoritative list of predefined constants.

---

### 25.3 Beyond Platform Selection — Where `when` Earns Its Keyword

`when` is a general compile-time tool, not just an OS switch. It is used to:

* **select platform/target code** — OS, architecture, word size, endianness
* **gate build-mode code** — debug-only assertions, logging, and instrumentation that vanish from release binaries at zero cost
* **toggle features** — build flags with no runtime check and no linker symbol
* **specialize generics with zero cost** — choose an implementation from a compile-time property of a type parameter, with no runtime branch:

```bestie
fun store<T>(x: T) {
    when (sizeOf(T) <= 16) {
        inlineSmall(x)      // small values: pass/keep by value
    } else {
        boxLarge(x)         // large values: heap-back
    }
}
```

* **validate configuration at compile time** — combine with `const` to reject impossible build configurations before a binary is ever produced.

This breadth — spanning platform, build, features, and generic specialization — is why `when` is a core keyword rather than a niche directive: it is the single, uniform way to express *any* decision the compiler can settle, and every such decision it settles is a runtime branch that never has to exist.

---

### 25.4 `when` as Expression

`when` can be used as an expression when both branches produce the same type:

```bestie
val pageSize: int = when (target.os == "windows") { 4096 } else { 8192 }
```

The unchosen branch is completely eliminated. The result is a compile-time constant when both sides are constant. This mirrors `if` and `switch`, which are also expressions (§12) — the difference is purely *when* the choice is made: `when` at compile time, `if`/`switch` at runtime.

---

### 25.5 `when`/`else when`/`else`

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

### 25.6 `when` vs runtime `if` / `switch`

`when` is **compile-time**; `if` and `switch` (§12) are **runtime**. They are not interchangeable:

| | `when` | `if` / `switch` |
| --- | --- | --- |
| Evaluated | Compile time | Runtime |
| Condition | Must be compile-time constant | Any runtime expression |
| Losing branch | Erased — never compiled or type-checked | Compiled, type-checked, present in binary |
| Runtime cost | Zero — nothing emitted | A branch / jump instruction |

Use `switch`/`if` for decisions on runtime values. Use `when` for decisions fixed at compile time — it is faster (no branch) and it can guard code that would not compile on other targets.

**Compiler diagnostic.** When a runtime `switch` or `if` branches solely on compile-time constants (e.g. `switch (target.os) { ... }`), the compiler emits a **warning** recommending `when`, because the decision is knowable at compile time and the runtime form needlessly compiles every branch and emits a live branch instruction:

```
warning: 'switch' branches only on compile-time constants (target.os) —
         use 'when' to resolve this at compile time and eliminate dead branches
```

The warning is advisory, not an error: a runtime `switch` over `target.os` is still valid, it is simply never the intended tool for a compile-time decision.

---

### 25.7 Rules

* `when` conditions must be **compile-time evaluable** — runtime expressions are a compile-time error
* The rejected branch is **completely eliminated** — it is not type-checked as dead code
* `when` is not a runtime `if` — it does not produce branches in the binary
* `when` may appear at module level (top-level declarations), inside functions, as expressions, and around class members and function definitions
* Nesting is allowed
* A runtime `switch`/`if` that branches only on compile-time constants triggers the advisory "use `when`" warning (§25.6)

---

## 26. Stability

* Core is sealed
* Backward compatibility is mandatory
* Experimental features require flags
* Higher layers evolve independently

---

## 27. Cited Symbols (Frozen Names)

Core may **name** a type or method that is declared in `bestie.lib` or `bestie.api` — the same way C's `sizeof` yields a `size_t` that is declared in `stddef.h`. Doing so does not move that symbol into core, and it does not make the package part of core.

What it does do is **freeze the name**. A symbol that a normative core rule depends on cannot be renamed or removed while that rule exists, because renaming it would change what a keyword or an operator means. This is the mechanism behind `platform.md` §12: citation is what separates the parts of std-lib that are permanent from the parts that are freely deprecatable.

### 27.1 The list

| Cited symbol | Cited by | Declared in |
| ------------ | -------- | ----------- |
| `iterator()` | §13 — `for/in` desugaring | `bestie.lib.patterns` (`Iterable<T>`) |
| `next(): T ?` | §13 — `for/in` desugaring | `bestie.lib.patterns` (`Iterator<T>`) |
| `add` `sub` `mul` `div` `mod` `neg` | §15.3 — operator lowering | `bestie.lib.utilities` |
| `addAssign` `subAssign` `mulAssign` `divAssign` `modAssign` | §15.3 — compound assignment lowering | `bestie.lib.utilities` |
| `get(index)` / `set(index, value)` | §15.3 — `a[i]` and `a[i] = v` | `bestie.lib.utilities` |
| `equal(other): bool` | §15.3, §15.4 — `==` and `!=` | `bestie.lib.utilities` (`Equable<T>`) |
| `compareTo(other): int` | §15.3 — `<` `<=` `>` `>=`; §9.4 — `range<T>` bounds | `bestie.lib.utilities` (`Comparable<T>`) |
| `hash(): int` | §5.4 — default `map` literal inference | `bestie.lib.utilities` (`Hashable<T>`) |
| `list<T>` | §5.3, §5.4 — literal type annotation | `bestie.lib.collections` |
| `map<K,V>` | §5.3, §5.4 — default inference for `{k: v}` literals | `bestie.lib.collections` |
| `option<T>` · `Present` / `Not_Present` | §21, §24.2 — the named form of `T ?` | `bestie.lib.utilities` |
| `result<T,E>` · `Ok` / `Err` | §21, §24.2 — the named form of `T ! E` | `bestie.lib.utilities` |

### 27.2 What freezing does and does not mean

**Frozen:**

* The symbol's **name** and the **shape core relies on** — a parameter list, a return type, a method's meaning.
* Removing it, renaming it, or changing that shape is a **language break**, subject to §6 versioning in `platform.md`, not to the owning package's evolution rules.

**Not frozen:**

* Everything else about the declaring protocol or type. `Iterable<T>` may gain methods; `list<T>` may gain variations and operations; `Comparable<T>` may grow helpers. Additive change is always allowed.
* Any symbol **not** on this list. `Arena`, `FixedBuffer`, `matrix<T>`, `StringBuilder`, `Channel<T>`, every algorithm and every codec — none is cited by a core rule, so all of them may be deprecated and removed under the ordinary std-lib policy.

That asymmetry is the point. `ptr<T>` and `Iterator.next()` are permanent because core depends on them by name. `Arena` is not, and can be dropped in a later release without touching the language.

### 27.3 Changing the list

* **Adding a citation** is a core change: it converts a previously deprecatable symbol into a permanent one, and requires a `lang` version bump.
* **Removing a citation** is also a core change, but it *un-freezes* the symbol — after which the owning package may deprecate it normally.
* An **illustrative mention** is not a citation. A protocol that appears only in an example (`fun <T impl Serializable> …`) is not frozen; only symbols this table lists are. When a core rule starts depending on a symbol, it must be added here in the same change.

### 27.4 Compiler obligation

The compiler resolves every cited symbol from the declaring package with no import required at the use site: `for (x in xs)` compiles without `import bestie.lib.patterns`, and `a + b` compiles without `import bestie.lib.utilities`. The import is needed only to name the protocol yourself — for example to write `impl Addable<Vec2>`.

A cited symbol that is missing or has the wrong shape is a **toolchain error**, not a user error:

```
error: standard library is incompatible — 'Iterator.next' not found or has an unexpected signature
```

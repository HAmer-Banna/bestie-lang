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

// or use option<T> when absence is intentional
val result: option<int> = if (cond) option.Present(42) else option.None
```

Rules:

* No implicit zero-initialization of locals — the programmer is always in control
* Conditional initialization requires either a guaranteed `else` branch or `option<T>`
* The same rule applies inside loops — a binding declared before a loop must be assigned before the loop can read it
* `val` bindings assigned exactly once satisfy definite assignment; assigning a second time is a compile-time error

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

Bestie includes exactly one collection in the core language: `array<T>`.

`array<T>` is built-in and available without `import`.
It provides fixed-capacity contiguous storage, deterministic layout, and index-based access.

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
| `{k: v, k: v, ...}` | `map<K,V>.hash` | `val m = {1: "a", 2: "b"}` → `map<int,str>.hash` |

`{1, 2, 3}` without a type annotation is always an `array<int>`, never a `list` or `set`. This is intentional: the default is the most efficient, lowest-overhead form.

#### Explicit disambiguation

When the desired type differs from the default, an explicit type annotation on the binding overrides inference:

```bestie
val a = {1, 2, 3}                       // array<int>   — default
val b : list<int>    = {1, 2, 3}        // list<int>    — explicit
val c : set<int>     = {1, 2, 3}        // set<int>     — explicit (deduplicates)
val d : deque<int>   = {1, 2, 3}        // deque<int>   — explicit

val e = {1: "a", 2: "b"}               // map<int,str>.hash — default
val f : map<int,str>.linked = {1: "a"} // map<int,str>.linked — explicit variation
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
* Without a type annotation, `{k: v, ...}` **always** resolves to `map<K,V>.hash`
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

#### What the Compiler Does With Range Information

This is where range constraints create binary-level differences. Every piece of range information is a fact the compiler exploits without the programmer doing anything further.

**1. Single validation point — no repeated checks**

The check happens once at construction. Every subsequent use of the value is bounds-check-free. A `Score` passed through ten functions never re-validates.

**Object file impact:** Eliminates a `cmp + jae/jb` pair at every use. In a hot loop over a list of `Score` values, this removes N branches for N iterations.

---

**2. Array bounds check elimination**

When a range-constrained value is used as an index into a collection whose size matches or exceeds the range, the bounds check is eliminated entirely:

```bestie
val table: list<str>[101]   // indices 0..=100
val s: Score                // guaranteed 0..=100

val name = table[s]         // ✅ no bounds check emitted — range ⊆ [0..=100]
```

**Object file impact:** Removes the `cmp + conditional-branch` before the load instruction. In SIMD loops over indexed lookups, this also unblocks vectorization (bounds checks are a barrier to auto-vectorization).

---

**3. Enum niche optimization**

When a range-constrained type is used inside an `enum` payload, the unused bit patterns of the underlying type are available to the compiler to encode the discriminant — eliminating the tag field entirely.

```bestie
type Score as int32 in 0..=100
// valid bit patterns: 0..=100
// unused bit patterns: 101..=2^31-1 (over 2 billion niches)

enum GameResult {
    Win(Score)    // compiler uses bit patterns 101+ to encode the Lose/Draw discriminants
    Lose
    Draw
}
// sizeof(GameResult) == sizeof(int32) — no separate tag field
```

**Object file impact:** Saves 4–8 bytes per `enum` instance (the eliminated tag field + its padding). In a `list<GameResult>`, every element is 4 bytes instead of 8 — twice the elements fit in a cache line.

---

**4. Arithmetic overflow elimination**

When the compiler can prove that arithmetic on range-constrained values stays within the valid range of the result type, overflow checks are eliminated:

```bestie
type Byte as uint8 in 0..=200

val a: Byte
val b: Byte
val sum = (a as uint16) + (b as uint16)   // max = 400, fits in uint16 — no overflow possible
```

**Object file impact:** Eliminates `jo`/`jno` (overflow flag checks) or the `ADDS`/`ADDV` with branch instructions on ARM. In arithmetic-heavy loops, this is a measurable throughput improvement.

---

**5. Smaller internal storage (struct packing)**

When a range-constrained type's range fits within a smaller underlying type, the compiler may use smaller storage **within struct layouts** while preserving the declared type in function signatures and operations.

```bestie
type WeekDay as int32 in 0..=6
// range fits in 3 bits / 1 byte — compiler uses uint8 storage in structs
```

The declared type (`int32`) governs arithmetic and ABI. The compiler chooses the minimum storage in struct fields transparently.

**Object file impact:** A struct with four `WeekDay` fields uses 4 bytes instead of 16. Better cache density.

---

**6. Float constraints eliminate NaN/Inf checks**

A `float64 in 0.0..=1.0` is guaranteed finite and in range. The compiler can:
- Skip `isNaN` / `isInfinite` guards before using the value
- Apply SIMD strategies that are only valid for finite, bounded floats
- Use approximation instructions (`RCPPS`, `RSQRTPS`) without correctness concerns

---

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

The Bestie compiler is **obligated** to use the minimum valid representation for every type. This is not a quality-of-implementation optimization — it is a spec requirement. The compiler must never use more bits or bytes than the type's value set requires.

This applies everywhere the compiler has type-level information: enum discriminants, payload types, struct fields, bool values, characters, sealed hierarchies, and any other composite type.

---

#### Enum Discriminant Sizing

The discriminant (tag field) of an enum uses the minimum integer type needed to encode all variants. No enum pays for a 4-byte tag when 1 byte is sufficient.

| Variant count | Discriminant storage | Bytes |
| ------------- | -------------------- | ----- |
| 2             | `uint1` (stored as `uint8`) | 1 |
| 3 – 256       | `uint8`              | 1     |
| 257 – 65 536  | `uint16`             | 2     |
| 65 537+       | `uint32`             | 4     |

```bestie
enum Direction { North, South, East, West }
// 4 variants → uint8 discriminant → 1 byte tag (not 4)
```

**Object file impact:** A struct containing a `Direction` field uses 1 byte for the tag, not 4. After field reordering, this tag may share a padding gap that already existed — net cost zero.

---

#### Niche Optimization — Discriminant-Free Enums

When a payload type has **invalid bit patterns** (values the type can never legally hold), the compiler uses those patterns to encode discriminant values — eliminating the tag field entirely.

The compiler performs niche analysis automatically. No annotation required.

**Types with known niches:**

| Type | Valid bit patterns | Available niches |
| ---- | -------------- | ---------------- |
| `bool` | `0`, `1` | 254 niche patterns (2–255) |
| `char` | `0..=0xD7FF`, `0xE000..=0x10FFFF` | All surrogate and out-of-range codepoints |
| `ptr<T>` (from `own`) | Any non-zero address | Zero address (0x0) — 1 niche |
| `uint8 in 0..=200` | `0..=200` | 55 niche patterns (201–255) |

> **Note:** Bestie has no `null` value — the zero address (0x0) is an **internal bit pattern** used only by the compiler as a niche slot. It is never a language-level value, never expressible in Bestie source code, and never returned from safe Bestie functions. The niche mechanism is entirely transparent to the programmer.

```bestie
enum MaybeChar {
    Some(char)   // char has invalid codepoints — compiler uses one as the None discriminant
    None
}
// sizeof(MaybeChar) == sizeof(char) == 4 bytes. No separate tag field.

enum NonNullPtr {
    Live(own Foo)   // own Foo is always a valid address — compiler uses 0x0 as the Dead discriminant
    Dead
}
// sizeof(NonNullPtr) == sizeof(ptr) — no tag byte
```

**Niche selection:** The compiler assigns niches greedily — most structurally constrained variant first. For enums with multiple payloads, the payload with the most available niches is analyzed first.

**Object file impact:** Eliminates 1–8 bytes per enum instance (the tag field and its padding). In a `list<MaybeChar>`, every element is 4 bytes instead of 8 — twice the density per cache line.

---

#### Bool Representation

`bool` is always stored as 1 byte. The values `0` and `1` are the only valid bit patterns. The compiler treats any use of a `bool` value as known to be in `{0, 1}` — no masking, no widening checks needed when promoting to a wider integer type.

Multiple `bool` fields in a struct are **not** automatically bit-packed (bit-packing has read-modify-write cost on writes). They benefit from field reordering — consecutive `bool` fields are grouped, consuming the minimum number of bytes with no padding between them.

---

#### `char` Representation

`char` is a 32-bit Unicode scalar. The valid range is `0..=0xD7FF` and `0xE000..=0x10FFFF`. The surrogate range (`0xD800..=0xDFFF`) and values above `0x10FFFF` are permanently invalid. The compiler exploits these as niches in enum payloads without any programmer action.

---

#### Sealed Class Type Tag

A `sealed` class hierarchy has a finite, compile-time-known set of concrete types. The compiler uses a **minimum-size type tag** — the same sizing rule as enum discriminants — instead of a vtable pointer.

```bestie
sealed class Shape permits Circle, Rectangle, Triangle
// 3 implementors → uint8 type tag → 1 byte (not a pointer-sized vtable)
```

Dispatch through a sealed type uses a `switch` on the tag byte, not an indirect call through a vtable. This eliminates the vtable pointer load and the indirect branch.

---

#### General Principle

The compact representation guarantee extends to every place the compiler stores a discriminant, tag, flag, or type identifier:

* Enum discriminants — minimum integer size
* Niche optimization — zero bytes when payload has available invalid patterns
* Sealed type tags — minimum integer size, not pointer-sized
* `bool` fields — 1 byte, grouped by field reordering, no hidden padding
* `char` — niches available to enclosing enum

The programmer does nothing to enable this. It is the compiler's obligation. The only exception is types marked `@layout(stable)` for FFI compatibility — those are laid out exactly as declared and exempt from all reordering and compaction.

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

If a value does not have a natural empty representation, `option<T>` must be used.

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
  case 1 => 10
  case 2 => 20
  case _ => 0
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

`defer` schedules a statement to execute at the **end of the enclosing scope**, regardless of how the scope exits — normal return, early return, or error propagation through `try`.

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
* `defer` body cannot use `return` or `try`
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
* Pattern matching is **compile-time lowered** to comparisons, branches, and sealed-tag tests where needed — no reflection and no runtime dispatch tables
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

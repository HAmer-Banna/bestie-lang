# Bestie Core Types

This document defines the **programmer-facing surface** of Bestie's built-in core types.
It answers questions such as:

* Which types are built into the language?
* Which operators are valid on them?
* Which methods appear directly on values such as `5`, `'A'`, `"hello"`, or `0..10`?

Bestie core types behave like **zero-cost value classes** from the programmer's point of view:

```bestie
val a: str = 5.toStr()
val b: str = true.toStr()
val c: str = 'A'.toStr()
```

These method calls do **not** imply heap allocation, object headers, or runtime dispatch.
They are compile-time known operations lowered directly by the compiler.

`ptr<T>` is intentionally excluded from this document.
Raw pointer semantics are defined in `memory.md`.

---

## 1. Design Principles

Core type operations follow these rules:

* Built-in types are always available without import
* Methods on built-in value types are statically resolved
* No implicit numeric widening or narrowing exists
* Conversion is always explicit
* Operators never hide allocation
* Higher-level domain behavior belongs in `std-lib`, not in core

As a result, core methods are intentionally small, predictable, and close to the machine model.

---

## 2. Primitive Numeric Types

Bestie includes these primitive numeric families:

| Family | Types |
| ------ | ----- |
| Signed integers | `int8`, `int16`, `int32`, `int64`, `int` |
| Unsigned integers | `uint8`, `uint16`, `uint32`, `uint64`, `uint` |
| Byte alias | `byte` (`uint8`) |
| Floating point | `float32`, `float64` |

All numeric primitives are **value types**.
They copy by value, have deterministic layout, and require no deallocation.

### 2.1 Common Numeric Conversions

All numeric types support explicit conversion methods.
Method naming follows the target type:

* `toInt8()`, `toInt16()`, `toInt32()`, `toInt64()`, `toInt()`
* `toUInt8()`, `toUInt16()`, `toUInt32()`, `toUInt64()`, `toUInt()`
* `toFloat32()`, `toFloat64()`
* `toByte()` when converting to the `byte` alias
* `toStr()` for textual conversion

Rules:

* Widening conversions are total
* Narrowing conversions are checked
* A checked narrowing conversion uses `try` when runtime validation is required
* `@trusted` may suppress runtime checks, with the same responsibility model as other `@trusted`/`ptr` low-level operations — correctness is the programmer's to uphold

Example:

```bestie
val a: int64 = 5.toInt64()
val b: str   = 5.toStr()
val c = try userInput.toUInt8()
```

Method syntax and `as` casting are equivalent explicit conversion surfaces.

---

### 2.2 Signed Integers

Signed integer types are:

* `int8`
* `int16`
* `int32`
* `int64`
* `int` (pointer-sized)

#### Operators

Signed integers support:

* Arithmetic: `+`, `-`, `*`, `/`, `%`
* Unary negation: `-x`
* Comparisons: `==`, `!=`, `<`, `<=`, `>`, `>=`
* Bitwise operations: `&`, `|`, `^`, `~`, `<<`, `>>`
* Compound assignment: `+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `|=`, `^=`, `<<=`, `>>=`

#### Methods

| Method | Returns | Notes |
| ------ | ------- | ----- |
| `toStr()` | `str` | Decimal text by default |
| `toInt8()` … `toFloat64()` | target type | Explicit conversion (§2.1) |

Arithmetic and bit helpers (`abs`, `sign`, `isZero`, `countOnes`, `leadingZeros`, `trailingZeros`, …) are extension functions in `bestie.lib.math` — see §2.7.

Example:

```bestie
val n: int = -42
val s = n.toStr()          // core
val big = n.toInt64()      // core
```

---

### 2.3 Unsigned Integers and `byte`

Unsigned integer types are:

* `uint8`
* `uint16`
* `uint32`
* `uint64`
* `uint` (pointer-sized)
* `byte` (`uint8`)

`byte` is the idiomatic spelling when the value represents raw binary data rather than a small number.

#### Operators

Unsigned integers support:

* Arithmetic: `+`, `-`, `*`, `/`, `%`
* Comparisons: `==`, `!=`, `<`, `<=`, `>`, `>=`
* Bitwise operations: `&`, `|`, `^`, `~`, `<<`, `>>`
* Compound assignment forms of the above

#### Methods

| Method | Returns | Notes |
| ------ | ------- | ----- |
| `toStr()` | `str` | Decimal text by default |
| `toUInt8()` … `toFloat64()` | target type | Explicit conversion (§2.1) |

Bit helpers live in `bestie.lib.math` — see §2.7.

Example:

```bestie
val raw: byte = 0x41
val text = raw.toStr()     // core
```

---

### 2.4 Floating-Point Types

Floating-point types are:

* `float32`
* `float64`

Both use IEEE 754 semantics.

#### Operators

Floating-point values support:

* Arithmetic: `+`, `-`, `*`, `/`
* Unary negation: `-x`
* Comparisons: `==`, `!=`, `<`, `<=`, `>`, `>=`
* Compound assignment: `+=`, `-=`, `*=`, `/=`

#### Methods

| Method | Returns | Notes |
| ------ | ------- | ----- |
| `toStr()` | `str` | Explicit textual conversion |
| `toFloat32()` / `toFloat64()` / `toInt*()` | target type | Explicit conversion (§2.1) |
| `isNaN()` | `bool` | IEEE 754 NaN check |
| `isInfinite()` | `bool` | Positive or negative infinity |
| `isFinite()` | `bool` | True when neither NaN nor infinity |

The three classification predicates stay in core because `core/constants.md` §3.2 makes them normative: `T.NAN` is never equal to itself, so `.isNaN()` — not `== float64.NAN` — is the *only* correct test, and core must supply the operation it mandates.

Rounding and sign helpers (`abs`, `floor`, `ceil`, `round`, `trunc`) are extension functions in `bestie.lib.math` — see §2.7 — alongside `sqrt`, `pow`, trigonometry, and linear algebra.

---

### 2.5 `bool`

`bool` is a primitive logical value with exactly two states:

* `true`
* `false`

#### Operators

`bool` supports:

* Logical negation: `not`
* Logical conjunction: `and`
* Logical disjunction: `or`
* Equality and inequality: `==`, `!=`

`!`, `&&`, and `||` are not boolean operators. `!` is reserved for `T ! E` and overflow-trap (`+!`). See `lang.md` §15.

#### Methods

| Method | Returns | Notes |
| ------ | ------- | ----- |
| `toStr()` | `str` | `"true"` or `"false"` |
| `toInt()` | `int` | `1` or `0` |

---

### 2.6 `char`

`char` is a 32-bit Unicode scalar value.
It is not a UTF-16 code unit and not a byte.

#### Operators

`char` supports:

* Equality and inequality: `==`, `!=`
* Ordering: `<`, `<=`, `>`, `>=`

#### Methods

| Method | Returns | Notes |
| ------ | ------- | ----- |
| `toStr()` | `str` | Single-character string |
| `toInt32()` | `int32` | Unicode scalar value |

Classification (`isAscii`, `isDigit`, `isLetter`, …) is provided by `bestie.lib.strings`. Text-shaping behavior such as locale-aware casing, normalization, and segmentation belongs there too.

---

### 2.7 Numeric Helpers Live in `bestie.lib.math`

Core keeps exactly two things on a numeric primitive:

* **Operators** — `+`, `-`, `*`, `/`, `%`, comparisons, bitwise, compound assignment. The compiler lowers these directly (`lang.md` §15).
* **Explicit conversions** — `toStr()`, `toInt64()`, `toFloat32()`, and the rest of §2.1. These are semantic: they are the method spelling of the `as` casting rules, and core owns casting.

Everything else — `abs`, `sign`, `isZero`, `countOnes`, `leadingZeros`, `trailingZeros`, `floor`, `ceil`, `round`, `trunc` — is an **extension function** in `bestie.lib.math`:

```bestie
import bestie.lib.math

val n: int = -42
val a = n.abs()                    // lib extension, reads as a method
val bits = 0xFF00_FF00.countOnes() // lib extension
```

This is the same split core already applies to `str` (§3): core `str` has `byteSize()`, `char(i)`, and `chars()`; `substring`, `split`, `trim`, and `toInt()` are lib extensions.

**Why:** these helpers are the part of the numeric surface that will keep growing — `rotateLeft`, `nextPowerOfTwo`, `copySign`, `clamp`, `saturatingAbs`. A programmer who does not know Bestie has `countOnes` has learned nothing wrong about the language; a programmer who does not know Bestie has `data class` has. Freezing the growable half into core would mean every future addition is a core release, for no gain: an extension function is statically resolved and monomorphized, so `n.abs()` compiles to exactly the same instruction either way (`fp.md` §11.2).

The **operators** are the part that must never move, and they stay.

---

## 3. `str`

`str` is Bestie's built-in immutable UTF-8 string type.

`str` is a **value type** with immutable contents.
Copying a `str` never changes the visible semantics of the value.

### Operators

`str` supports:

* Concatenation: `+`
* Equality and inequality: `==`, `!=`
* Lexicographic comparison: `<`, `<=`, `>`, `>=`
* Byte indexing: `s[i] -> byte`
* Negative byte indexing: `s[-i] -> byte` — `s[-1]` is the last byte
* Byte slicing: `s[lo..hi] -> slice<byte>` — a borrowed, zero-copy view; see §6

### Methods

| Method | Returns | Notes |
| ------ | ------- | ----- |
| `toStr()` | `str` | Identity conversion |
| `isEmpty()` | `bool` | Zero bytes |
| `byteSize()` | `int` | UTF-8 byte length |
| `char(index: int)` | `char` | Unicode scalar at codepoint index |
| `chars()` | iterator over `char` | Iterates over Unicode scalars |
| `bytes()` | iterator over `byte` | Iterates over raw UTF-8 bytes |

Rules:

* `s[i]` is raw byte access, not character access
* `s[-i]` is end-relative byte access — `s[-1]` is the last byte; sugar for `s[s.byteSize() - i]`
* `s.char(i)` is the explicit Unicode-aware path
* Bestie intentionally does not overload a single ambiguous `length()` meaning in core

> **Why `s[i]` returns `byte` and not `char`.** `str` is stored as UTF-8, which is **variable-width** (a codepoint is 1–4 bytes). Byte access at a byte offset is therefore **O(1)** — a single memory read with no decoding. Indexing by *codepoint* (`s[i] -> char`) cannot be O(1) on a UTF-8 buffer: finding the i-th scalar requires decoding from the start, which is **O(n)**. Bestie refuses to hide that cost behind `[]`. Making `s[i]` return `char` would force either O(n) subscripting (a hidden cost) or a fixed-width 32-bit internal representation (4× memory and no zero-copy interop with byte buffers, I/O, and FFI). So `[]` stays the fast byte path, and codepoint access is explicit via `s.char(i)` / `s.chars()`. (A `char` is still only a Unicode scalar, not a grapheme cluster, so no integer subscript yields a user-perceived "character" anyway — that belongs in `std-lib/strings.md`.)

Example:

```bestie
val s: str = "Bestie"

val firstByte: byte = s[0]
val lastByte:  byte = s[-1]
val firstChar: char = s.char(0)
val empty: bool = s.isEmpty()
```

#### Slicing a `str`

Because `str` indexing is **byte-level**, slicing is byte-level too: `s[lo..hi]` returns a `slice<byte>` view (not a new `str`), with no copy. The bound forms match `array<T>` and `range<T>`:

```bestie
val s : str = "Bestie"

val head = s[0..3]    // slice<byte> over the bytes of "Bes"
val tail = s[-2..]    // slice<byte> over the last two bytes
```

`str` is immutable, so a `slice<byte>` over a `str` is always a **read-only** view and never risks the source mutating underneath it (see §6).

> A slice taken at an arbitrary byte boundary may split a multi-byte UTF-8 scalar. Core slicing does not validate this — it is raw byte access by definition. A codepoint-aware, validity-checked substring that returns a `str` is a **std-lib** concern, alongside searching, splitting, normalization, and case conversion.

#### String parsing and text operations (std-lib)

Core `str` deliberately omits **parsing** (`str → int`, etc.) and **higher-level text operations** (`substring`, `split`, `trim`, case conversion, search). These are provided as **extension functions** in `std-lib/strings.md`, so they read as methods (`s.toInt()`, `s.substring(0, 3)`) without enlarging the core type:

```bestie
import bestie.lib.strings

val n   = s.toInt() catch |e| { 0 }   // fallible parse — returns int ! ParseError
val sub = s.substring(0, 3)           // codepoint-aware, returns an owned str
```

Parsing is fallible (it returns `T ! ParseError`), which is exactly why it lives outside core alongside the other text utilities, rather than next to the total, infallible numeric `toStr()`.

---

## 4. `tuple`

Tuples are built-in positional value aggregates.

Example:

```bestie
val pair: (int, str) = (7, "days")
```

### Properties

* Tuples are value types
* Tuple layout is compile-time known
* No heap allocation is required
* Tuples are positional, not nominal

### Operations

| Operation | Meaning |
| --------- | ------- |
| `t.0`, `t.1`, ... | Positional field access |
| `val a, b = t` | Destructuring |
| `return x, y` | Tuple return shortcut |

Example:

```bestie
fun divMod(x: int, y: int): (int, int) {
    return x / y, x % y
}

val q, r = divMod(10, 3)
```

Tuples do not define domain behavior.
If named meaning, invariants, or methods are needed, use a class or a `data class`.

---

## 5. `array<T>`

`array<T>[N]` is Bestie's built-in **static array**: exactly `N` elements of `T`, stored contiguously, with `N` fixed at compile time. It is a C array — same layout, same cost — with an API that is easier to use than Java's.

```bestie
val primes : array<int>[] = {2, 3, 5, 7, 11}     // length 5, inferred from the literal
var buf    : array<byte>[4096] = array<byte>[4096].fill(0)

primes.length          // 5 — a compile-time constant
buf[0] = 0x7F
```

### 5.1 Length Is Part of the Type

`N` is a compile-time constant — a literal, a `const`, or a `target.*` constant (`constants.md`). It is part of the type: `array<int>[3]` and `array<int>[4]` are different types, exactly as `int[3]` and `int[4]` are in C.

There is **no runtime-length array**. A buffer whose length is known only at run time is a `list<T>` from `bestie.lib.collections`, viewed as a `slice<T>` with `ls[..]` wherever a function only needs the elements. That keeps `array<T>` honest: when you see one, its length, its layout, and its stack footprint are all known to the compiler.

`array<T>` without `[N]` is not a type on its own. It is written only where the length is inferred from a literal on the same line (`array<int>[] = {…}`) or in prose to mean "an array of any length". A function that accepts arrays of any length takes `slice<T>` (§5.10).

### 5.2 Construction — Every Element Is Initialized

An array is always **full**. Every one of its `N` elements holds a value from the moment it exists; there is no empty array, no `add()`, and no uninitialized element.

| Form | Meaning |
| ---- | ------- |
| `array<T>[] = {v1, v2, ...}` | Length inferred from the literal |
| `array<T>[N] = {v1, ..., vN}` | Length stated; the literal must have exactly `N` values |
| `array<T>[N].fill(v)` | All `N` elements set to `v` |
| `array<T>[N].from(f)` | Element `i` set to `f(i)` |

```bestie
val days    : array<str>[] = {"Mon", "Tue", "Wed", "Thu", "Fri"}
var counts  : array<int>[26] = array<int>[26].fill(0)
val squares : array<int>[10] = array<int>[10].from((i: int) => i * i)
```

Rules:

* A declaration with no initializer — `val a : array<int>[5]` — is a compile-time error. C leaves those elements as garbage; Java silently zeroes them. Bestie makes you say which value you mean.
* A literal whose count differs from a stated `N` is a compile-time error.
* `fill` and `from` cost nothing extra: `fill(0)` lowers to the same `memset` a C compiler emits, and `from` takes a non-capturing lambda that is inlined into the initializing loop. When the compiler can prove every element is overwritten before it is read, the fill is removed.

### 5.3 Operators

| Operation | Meaning |
| --------- | ------- |
| `arr[i]` | Element access — **panics** if out of bounds |
| `arr[-i]` | Access from the end — `arr[-1]` is the last element. Sugar for `arr[arr.length - i]` |
| `arr[i] = v` | Element assignment on a `var` array — **panics** if out of bounds |
| `arr[lo..hi]` | A borrowed `slice<T>` view over `[lo, hi)`; **no copy**. See §6 |
| `for (x in arr)` | Iterate over all elements |

When the index is a compile-time constant, the bounds check is resolved at compile time: an out-of-range constant index is a compile-time error, and an in-range one emits no check at all.

### 5.4 Negative Indexing

A negative index counts from the end. It is pure syntax — `arr[-i]` lowers to `arr[arr.length - i]`, and because `length` is a constant, that subtraction is folded away:

```bestie
var xs : array<int>[] = {10, 20, 30, 40, 50}

val last   = xs[-1]    // 50
val penult = xs[-2]    // 40
xs[-1] = 99            // xs → {10, 20, 30, 40, 99}
```

Out-of-range negative indices **panic**, identically to positive ones. `0` and `-0` are the same index.

### 5.5 Slicing

`arr[lo..hi]` produces a **`slice<T>`** — a borrowed, zero-copy view over a contiguous run of the array. The bound syntax matches `range<T>` (§7):

```bestie
val xs : array<int>[] = {10, 20, 30, 40, 50}

val a = xs[1..4]     // {20, 30, 40}   (half-open)
val b = xs[1..=3]    // {20, 30, 40}   (inclusive end)
val c = xs[2..]      // {30, 40, 50}
val d = xs[..2]      // {10, 20}
val e = xs[..]       // the whole array
val f = xs[-2..]     // {40, 50}
val g = xs[..-1]     // {10, 20, 30, 40}
```

A slice of a `var` array may be taken as `slice<var T>` to write through it. Out-of-range bounds panic; `lo > hi` panics. See §6.

### 5.6 Members

| Member | Type | Notes |
| ------ | ---- | ----- |
| `length` | `int` | The element count `N`. A compile-time constant, read like a field — not a method call |
| `get(index: int)` | `T` | Method form of `arr[i]` |
| `set(index: int, value: T)` | `void` | Method form of `arr[i] = v`; `var` arrays only |
| `a == b`, `a != b` | `bool` | Element-wise, when `T` is `Equable` and both have the same `N`. One `memcmp` for primitive elements |

That is the whole core surface. Searching, sorting, and copying between ranges are `bestie.lib.algorithms`; conversions to a growable collection (`toList()`, `toSet()`) are extension functions in `bestie.lib.collections` — the same split core applies to `str`.

**Why `length` and not `size()`.** In Bestie, *size* means bytes — `sizeOf(T)`, `str.byteSize()`. An array's element count is a different quantity, and it never changes, so it gets a different word and no call parentheses: `arr.length`, exactly as in Java and C#. Growable collections in `bestie.lib.collections` keep `size()`, because their count is a live, changing value and reading it is a real (if cheap) operation. The distinction is visible at the call site: `length` is fixed, `size()` can change.

### 5.7 Iteration

`array<T>` implements `Iterable<T>`, and the `for/in` loop over it lowers to a direct index walk with no call:

```bestie
val scores : array<int>[] = {91, 84, 77}

for (s in scores) {
    println(s.toStr())
}

for (i in 0..scores.length) {
    println("${i}: ${scores[i]}")
}
```

### 5.8 Layout and Performance

`array<T>[N]` is laid out as **exactly `N` elements, `strideOf(T)` apart** — `N * strideOf(T)` bytes, aligned to `alignOf(T)` (`memory-layout.md` §3). Nothing else is stored: no header, no length word (the length is in the type), no capacity, no pointer. This is byte-for-byte the layout of a C array `T a[N]`.

It is a value type, so it lives **inline** wherever it is declared — in a stack frame, as a field inside an object, or as an element of another array — and is never separately heap-allocated. An array in a heap-allocated `class` is part of that object's single allocation.

| Property | `array<T>[N]` |
| -------- | ------------- |
| Object header | ❌ none |
| Stored length | ❌ none — `N` is in the type |
| Identity | ❌ none |
| Separate heap allocation | ❌ never |
| Element access `arr[i]` | one multiply-add-load, plus a bounds branch the predictor learns immediately |
| Inheritance / virtual dispatch | ❌ not applicable |

Because it lives inline, a very large array declared as a local occupies the stack. The compiler warns when a single frame exceeds the target's stack-size guidance; large buffers belong in a `class` field or a `list<T>`.

### 5.9 Copying

An array is a value: assignment copies every element, exactly like a `value class`.

```bestie
var a : array<int>[] = {1, 2, 3}
var b = a              // b is an independent copy — Java needs a.clone()
b[0] = 99              // a is still {1, 2, 3}
```

That copy is `N * strideOf(T)` bytes. It is what you want for small fixed records (a 3-vector, an IPv6 address); for buffers, pass a view instead (§5.10).

### 5.10 Passing Arrays — No Decay

C silently converts an array to a pointer to its first element whenever it is passed or used in an expression, and the length is lost. **Bestie has no array-to-pointer decay.** Every form of passing is written:

| You want | Parameter type | Call | Cost |
| -------- | -------------- | ---- | ---- |
| Read the elements, any length | `slice<T>` | `f(arr)` | Two words: base + length |
| Write the elements, any length | `slice<var T>` | `f(arr[..])` on a `var` array | Two words |
| Every length, each compiled exactly | `array<T>[N]` with `const N` (`lang.md` §7.2) | `f(arr)` | One pointer for a `val` parameter (`memory.md` §6.1) — no copy |
| One exact length | `array<T>[8]` | `f(arr)` | One pointer for a `val` parameter — no copy |
| A private copy to modify | `var xs: array<T>[N]` | `f(arr)` | A real copy, because you asked for one |
| A pointer to the first element (C interop) | `ptr<T>` | `f(arr[0].address())` | One word |

```bestie
fun average(xs: slice<int>): float64 {
    var total = 0
    for (x in xs) { total += x }
    return (total as float64) / (xs.length as float64)
}

val scores : array<int>[] = {91, 84, 77}
val avg = average(scores)         // the array is viewed, not copied
```

**An array converts to a read-only `slice<T>` wherever one is expected.** The conversion is a view — it copies nothing and keeps the length — so it is not decay, and it needs no syntax. A *mutable* view is never implicit: passing `arr[..]` to a `slice<var T>` parameter is how a call site says the callee may write the caller's array.

**A `val` array parameter is never copied.** Parameters are `val` unless declared `var` (`memory.md` §6.1), and a `val` parameter of a value type larger than two machine words is passed as a pointer to the caller's storage. The semantics are still those of a copy — the callee cannot modify it — so the pointer is invisible. Only a `var` parameter, which the callee may modify, costs a real copy.

#### Not Go's Arrays

Go also puts the length in the array type, and its arrays are rarely used because of what came with that. Bestie keeps the layout and drops the problems:

| Go | Bestie |
| -- | ------ |
| No way to write one function for `[3]int` and `[8]int` — everyone falls back to slices | `fun <const N: int>` (`lang.md` §7.2): one source, compiled exactly for each length |
| Passing an array copies all of it, silently | A `val` parameter is passed by pointer; only `var` copies |
| `f(arr[:])` needed everywhere a slice is expected | An array is a `slice<T>` argument as written |
| No methods — `len()` and loops by hand | `length`, slicing, negative indexing, `for/in`, `==`, and every `bestie.lib.algorithms` function (`sort`, `stableSort`, `binarySearch`, `partition`, `fold`, `zip`, …), which take slices and iterables and so accept arrays directly |
| Arrays and slices are two different worlds | Every slice operation works on an array; an array is the storage, a slice is the view |

A slice parameter is C's `(T *p, size_t n)` pair, carried as one value that cannot fall out of sync. `ptr<array<T>[N]>` is C's pointer-to-array, `T (*p)[N]`, which C programmers rarely reach for because the decaying form is the default.

### 5.11 Multi-Dimensional Arrays

Multi-dimensional arrays nest the length brackets. `array<int>[R][C]` is `array<array<int>[C]>[R]`: `R` rows, each an `array<int>[C]`.

Because an array stores nothing but its elements, the nested form is **one flat, contiguous, row-major block** of `R * C` elements — identical to C's `int a[R][C]`, with no row pointers and no per-row length:

```bestie
var grid : array<int>[3][4] = array<int>[3][4].fill(0)    // 12 contiguous ints
val mat  : array<float64>[][] = {{1.0, 2.0}, {3.0, 4.0}}  // 2×2, inferred
val cube : array<byte>[4][4][4] = array<byte>[4][4][4].fill(0)

grid[1][2] = 42        // offset (1 * 4 + 2) * strideOf(int) — one address computation
grid.length            // 3 — rows
grid[0].length         // 4 — columns
```

Rules:

* **Every dimension is a compile-time constant**, so every row has the same length and the element offset is computed with constants. Any or all of them may be inferred from a literal (`[][]`), but only from a **rectangular** literal — rows of different lengths are a compile-time error.
* `fill` and `from` apply at the innermost element: `array<int>[3][4].fill(0)` sets all 12 ints. `from` on a two-dimensional array receives the row index and returns a row.
* Indexing chains: `grid[r]` is a row (`array<int>[4]`, a place, not a copy), and `grid[r][c]` an element. Each index is bounds-checked against a constant.
* `grid[r][..]` is a `slice<int>` over one row. `grid[..]` is a `slice<array<int>[4]>` over the rows — the exact equivalent of C's `int (*p)[4]` plus a row count, and the way to pass a matrix of any height to a function.
* **There are no jagged arrays.** Rows of different lengths are a list of lists — `list<own list<T>>` in `bestie.lib.collections` — which is a different structure with a different cost (§5.12).
* A matrix whose dimensions are known only at run time is one `list<T>` of `rows * cols` elements with `r * cols + c` indexing, or a matrix type from `bestie.lib.math` that does that for you.

### 5.12 Arrays vs `list<T>`, and Lists of Lists

`array<T>[N]` and `list<T>` share indexing, negative indexing, slicing, and `for/in`, so moving between them needs no relearning. They differ in what they are:

| | `array<T>[N]` | `list<T>` |
| - | ------------- | --------- |
| Kind | Core value type | `class` in `bestie.lib.collections` |
| Length | Compile-time constant, `arr.length` | Runtime, changes, `ls.size()` |
| Storage | Inline — no allocation of its own | Heap buffer owned by the list object |
| Grows | ❌ never | ✅ reallocates as needed |
| Copy on assignment | ✅ all elements | ❌ — it is an owned object; copying is explicit |

The difference matters most for nesting:

| Declaration | Memory | Access `x[i][j]` | Like |
| ----------- | ------ | ---------------- | ---- |
| `array<int>[R][C]` | One flat block of `R * C` ints | One address computation, one load | C `int[R][C]` |
| `list<own list<int>>` | An outer buffer of pointers to inner list objects, each pointing to its own buffer | Load the inner list, load its buffer, load the element | Java `int[][]` / `List<List<Integer>>`, minus the object headers |

`list<list<int>>` without a qualifier is a compile-time error: `list<int>` is a reference type, so the outer list must say whether it **owns** its rows (`own`) or **borrows** them (`ref`) — `std-lib/collections.md` §0. Unlike C++'s `vector<vector<int>>`, which stores each inner vector's header inline in the outer buffer, each inner `list` is its own object reached through a pointer. Use a list of lists for genuinely jagged or growing data; use a multi-dimensional array, or one flat `list<T>`, for a dense matrix.

### 5.13 Immutability

An array is a value type, so it follows the value-type rule (`immutability.md` §4): **the binding freezes the whole value.**

| Declaration | Rebind? | Mutate elements? | Storage |
| ----------- | ------- | ---------------- | ------- |
| `var arr : array<int>[3] = …` | ✅ | ✅ | inline |
| `val arr : array<int>[3] = …` | ❌ | ❌ | inline |
| `const arr : array<int>[] = {1, 2, 3}` | ❌ | ❌ | `.rodata`, literal only |

```bestie
val fixed : array<int>[] = {1, 2, 3}
fixed[0] = 9           // ❌ compile-time error — a val array is frozen, like a val value class
```

This is C's `const int a[3]` for a `val`, and a true compile-time table for a `const`. There is no `array<T>.immutable` and no `freeze()` — those exist for heap collections, whose binding freezes only a reference. Sharing a `val` or `const` array across threads needs no lock.

### 5.14 Coming from Java

| Java | Bestie |
| ---- | ------ |
| `int[] a = new int[5];` | `var a : array<int>[5] = array<int>[5].fill(0)` |
| `int[] a = {1, 2, 3};` | `val a : array<int>[] = {1, 2, 3}` — or just `val a = {1, 2, 3}` |
| `a.length` | `a.length` |
| `a.clone()` | `var b = a` |
| `Arrays.fill(a, 7)` | `a = array<int>[5].fill(7)` |
| `new int[3][4]` (array of row objects) | `array<int>[3][4]` — one flat block |
| `int[][]` with ragged rows | `list<own list<int>>` |
| `final int[] a` (elements still mutable) | `val a` (elements frozen too) |
| `ArrayIndexOutOfBoundsException` | panic — or a compile error for a constant index |
| Length chosen at run time | `list<T>` |

---

## 6. `slice<T>`

`slice<T>` is a built-in **borrowed view** over a contiguous run of elements. It is the result of slicing an `array<T>` (or, in `std-lib`, an array-backed `list<T>`), and a `slice<byte>` is the result of slicing a `str`.

A `slice<T>` is exactly two words — a base and a length — with **no heap allocation, no copy, and no ownership of the underlying storage**. Creating one is O(1). It is the one fat view in core: it cannot be stored, and it cannot outlive its source (`memory.md` §13).

```bestie
val xs : array<int>[] = {10, 20, 30, 40, 50}
val mid : slice<int> = xs[1..4]   // view of xs[1], xs[2], xs[3] — no copy
```

### Bound Forms

Slice bounds reuse the `range<T>` syntax (§7), so there is one consistent convention across the language:

| Form | Meaning |
| ---- | ------- |
| `xs[lo..hi]` | Half-open — indices `lo` up to but excluding `hi` |
| `xs[lo..=hi]` | Inclusive — indices `lo` through `hi` |
| `xs[lo..]` | From `lo` to the end |
| `xs[..hi]` | From the start up to `hi` (exclusive) |
| `xs[..]` | The whole source |
| `xs[-k..]` | Last `k` elements |
| `xs[..-k]` | Everything except the last `k` |

Out-of-range bounds **panic** (consistent with `xs[i]`); `lo > hi` **panics**. When bounds and size are compile-time known, the check is resolved at compile time.

### Operators

| Operation | Meaning |
| --------- | ------- |
| `s[i]` | Element access into the slice — **panics** if out of bounds |
| `s[-i]` | End-relative access — `s[-1]` is the slice's last element |
| `s[lo..hi]` | Re-slice — a narrower `slice<T>` view (still no copy) |
| `s[i] = v` | Write-through assignment — **only** on a mutable slice (`slice<var T>`) |
| `for (x in s)` | Iterate over the slice's elements |

### Methods

| Method | Returns | Notes |
| ------ | ------- | ----- |
| `length` | `int` | Number of elements in the view — read like a field, as on `array<T>` (§5.6) |
| `isEmpty()` | `bool` | True when `length == 0` |
| `copyFrom(src: slice<T>)` | `void` | On `slice<var T>` only: copies `src` element-wise into this view — **panics** if the lengths differ. One `memmove` |

An owned, escapable copy of a slice is a `list<T>` — `toList()`, an extension function in `bestie.lib.collections`. There is no `toArray()`: an array's length is a compile-time constant and a slice's is not. To fill an array from a slice, write `arr[..].copyFrom(s)`.

A `slice<T>` implements `Iterable<T>`, so `for/in`, indexing, negative indexing, and re-slicing all behave exactly as they do on `array<T>`. Element access is a single contiguous load — pointer arithmetic, no indirection.

### View Rules

`slice<T>` is the one fat view with a compiler-checked lifetime (`memory.md` §13). It is not `ref` and not `ptr<T>`.

* It **cannot outlive its source** and **cannot be stored in a field**.
* It may be passed into calls. Returning a slice is valid only when it is a subview of a slice parameter (the result cannot outlive that parameter's source).
* While a `slice<T>` is alive, the source **cannot be mutated in a way that would move or free its storage** — for example, a `list<T>` cannot grow/reallocate while a slice views it. The compiler rejects such use.

### Read vs Mutable Slices

| Form | Meaning |
| ---- | ------- |
| `slice<T>` | Read-only view — elements may be read, not written |
| `slice<var T>` | Mutable view — `s[i] = v` writes through to the backing storage |

A `slice<var T>` is exclusive: while it is alive, no other access to the overlapped region is permitted. A read `slice<T>` may coexist with other read views of the same region.

> **Why `slice<var T>` and not `slice<const T>`.** `ptr<T>` and `slice<T>` mark write-through on opposite defaults, and that is deliberate rather than an inconsistency. A `ptr<T>` exists to reach and change memory, so mutability is its default and `const` is the restriction (`memory.md` §8.4.1 — this also mirrors C's four pointer forms exactly, which is the point of that spelling). A `slice<T>` exists to *look at* a run of elements, so read-only is its default and `var` is the escalation. Each type's default is the one it is for, and the marker is always the departure from that default. Both use a qualifier Bestie already has; neither introduces a third spelling.

### Immutable Sources

Slicing immutable storage is always safe and never requires a copy:

* A slice over a `str`, or over a `val` or `const` array, is inherently read-only.
* Because the source cannot mutate in place, the "no mutation while borrowed" rule is satisfied for free — the only remaining constraint is lifetime (the slice must not outlive the source).

### Class Kind and Performance

`slice<T>` has **value-class semantics** — no object header, no vtable, no identity, no heap. It is a fat pointer (base + length) passed in registers. This is what keeps slicing zero-cost: taking a slice is two register writes, and indexing through it is the same single load as indexing the underlying array.

> What slicing does **not** do: it never copies, never allocates, and never owns. To obtain an owned, escapable copy, call `.toList()` with `bestie.lib.collections` imported, or copy into an array you own with `copyFrom`. Only contiguous sources produce a `slice<T>`; non-contiguous collections are covered in `std-lib/collections.md`.

---

## 7. `range<T>`

`range<T>` is a built-in contiguous range value.

Example:

```bestie
val r1 = 0..10
val r2 = 0..=10
val r3 = 'a'..='z'
```

### Properties

* `range<T>` is a value type
* Constant ranges are compile-time resolved
* Runtime ranges are stack values
* `range<T>` implements `Iterable<T>`
* `T` must implement `Comparable`

### Operators and Syntax

| Syntax | Meaning |
| ------ | ------- |
| `a..b` | Exclusive upper bound |
| `a..=b` | Inclusive upper bound |
| `for (x in r)` | Iterate through the range |

### Methods

| Method | Returns | Notes |
| ------ | ------- | ----- |
| `length` | `int` | Number of values in the range — read like a field, as on `array<T>` (§5.6) |
| `contains(value: T)` | `bool` | Membership test |
| `mid()` | `T` | Midpoint when meaningful for `T` |
| `isEmpty()` | `bool` | Empty or reversed range |

Example:

```bestie
val r: range<int> = 0..100

val n = r.length
val m = r.mid()
val ok = r.contains(42)
```

---

## 8. Derived Core Type Forms

Bestie also has two important **type forms** built on top of the core type system.

### 8.1 Newtypes

`type X as Y` creates a distinct type with the same representation as `Y`.

```bestie
type UserId as int64
```

Properties:

* Distinct from the underlying type
* Zero runtime overhead
* Inherits the underlying type's operators and methods
* May define additional protocols or behavior

Example:

```bestie
type Meters as float64

val d: Meters = 12.5 as Meters
val s: str = d.toStr()
```

### 8.2 Range-Constrained Types

`type X as Y in range` creates a distinct type with an enforced invariant.

```bestie
type Port as uint16 in 1..=65535
```

Properties:

* Construction is checked once
* Use sites do not re-check the invariant
* Inherits the underlying type's operators and methods after successful construction

Example:

```bestie
type Score as int32 in 0..=100

val s = try (userInput.toInt32() as Score)
val text = s.toStr()
```

---

## 8.3 Optional types — `T ?`

`T ?` is **core syntax**. It is the type of a value that may be absent. It is not a second null; there is no implicit empty state on other types.

```bestie
fun find(id: int): User ?
fun connect(host: str, port: int ?)
val token: str ?
```

This syntax is sealed with the rest of core, and it is the **only** spelling. There is no named `option<T>` type: a second vocabulary for one representation would mean two ways to write every optional signature, and the whole surface below needs no import.

**The complete surface:**

* Signatures, parameters, and fields spelled `T ?`
* On `fun f(): T ?`: `return value` is present; a bare `return` is absent
* `absent` — the absent value in expression position, for fields and assignments
* `if (val x = …)` binds only when present (`fp.md` §3)
* `else` supplies a fallback or diverts (`fp.md` §3.3)
* FFI: C `NULL` maps to `ptr<T> ?` (`foreign.md`)

```bestie
class Session {
    userId: int
    token: str ?

    init(userId: int) {
        this.userId = userId
        this.token = absent
    }

    fun authenticate(tok: str) {
        this.token = tok        // a T value is accepted where T ? is expected
    }
}
```

`absent` is a contextual keyword valid only where a `T ?` is expected. It is not a null literal: it has no type of its own, cannot be compared with `==`, cannot be stored in a `T`, and cannot be dereferenced. Presence needs no constructor — a `T` is accepted wherever a `T ?` is expected, exactly as `return value` already works.

Default parameters (`x: int = 0`) are a different tool: the caller may omit the argument and the compiler fills a compile-time constant. `x: int ?` means the value may be absent at runtime. Both are valid; they are not interchangeable.

`?` binds to the immediately preceding type: `list<str ?>` is a list of optional strings; `list<str> ?` is an optional list.

---

## 8.4 Error unions — `T ! E`

`T ! E` is **core syntax**. `E` is an error set (see `exceptions.md`). This is recoverable failure, not absence.

```bestie
fun parse(s: str): int ! ParseError
```

As with `T ?`, this is the only spelling — there is no named `result<T, E>` type.

**The complete surface:**

* Signatures spelled `T ! E`
* `try` / `catch` at the call site (`exceptions.md` §2.3–2.4)
* `return ErrorSet.Variant` to fail explicitly (`exceptions.md` §2.5)
* Type arguments that are “a value or an error” — `Channel<int ! WorkError>`

Matching on a failure matches the **error set** directly, inside a `catch` — there is no `Ok` / `Err` wrapper to destructure:

```bestie
try connect() catch |e| switch (e) {
    case NetworkError.Timeout => retry()
    case NetworkError.Refused => giveUp()
}
```

Function signatures should use `T ! E`. Do not invent a second error API.

A function may not use `?` and `!` interchangeably for the same idea. Absence and failure are different: use `T ?` for “no value”, `T ! E` for “this failed”. Writing one where the other is meant — `parse(s: str): int ?`, hiding *why* it failed — is the mistake this rule exists to prevent.

#### Stacking `?` and `!`

`T ? ! E` is **legal**. `T ! E ?` is not.

A function returning `T ? ! E` has three outcomes: a value, a clean absence, and a failure. The value slot holds `T ?` and the error slot holds `E` — these are already separate registers in the `!` ABI (`exceptions.md` §3.6), so the form costs **nothing**: no third register, no extra branch, no allocation. The shape the ABI already has is exactly the shape this needs.

```bestie
fun next(): Path ? ! FsError        // an entry, end of directory, or a read error
fun lookup(k: Key): Row ? ! DbError // a row, a miss, or a query failure
```

**Binding.** `?` binds to the value and `!` is outermost, which follows from the existing rule that `?` binds to the immediately preceding type (§8.3). So `T ? ! E` reads as "*an optional `T`, or an error `E`*". The reverse — an optional error union — is meaningless and is a compile-time error:

```bestie
fun f(): int ? ! IoError    // ✅ a value, or nothing, or a failure
fun g(): int ! IoError ?    // ❌ 'T ! E ?' is not a type
```

**Unwrapping composes with no special case.** `try` handles the failure, `else` handles the absence, and each keeps its ordinary meaning:

```bestie
val entry = try it.next() else { break }
```

`try` propagates the `FsError` to the caller; `else` catches the clean end. A `try` on a `T ? ! E` yields `T ?`, which is then unwrapped like any other optional (`fp.md` §3.3).

**Why permit it.** The forcing case is a fallible iterator: `bestie.api.fs`'s `DirIterator.next()` must express the next entry, a clean end of directory, and a read error, and collapsing any two loses information. Every workaround under a strict reading is worse — an `End` error variant misrepresents normal termination as failure, a separate `hasNext()` needs lookahead and races, and a stateful `error()` checked after the loop is the `feof`/`ferror` model Bestie rejects elsewhere.

The deeper reason is that refusing the form does not remove the third state; it forces the author to *encode* it, badly. That is the same argument as the no-null guarantee (`fp.md` §4), one level up: absence should be in the type, not smuggled into a sentinel.

**Do not use it to dodge a decision.** `T ?` and `T ! E` remain different concepts, and the rule in this section still holds — `parse(s: str): int ?` is wrong because it hides *why* parsing failed. `T ? ! E` is for operations where absence and failure are **both genuinely possible and genuinely different**. A function whose absence case is really a failure should say so with `T ! E`.

For how `for/in` consumes a fallible iterator, see `lang.md` §13.

---

## 9. Summary

Bestie's core type surface is intentionally compact:

* Primitive numeric values behave like zero-cost value classes
* `bool`, `char`, and `str` expose explicit core operations only
* `tuple`, `array<T>`, `slice<T>`, `range<T>`, `T ?`, and `T ! E` are first-class core forms, each with exactly one spelling
* `array<T>[N]` is a C array: static length in the type, always fully initialized, flat row-major in every dimension, no decay to a pointer, and `arr.length` rather than `size()`
* Indexing is uniform: `[i]` panics out of bounds, `[-i]` counts from the end, `[lo..hi]` slices — the same on `array<T>`, `str`, and (in std-lib) array-backed `list<T>`
* `slice<T>` is a borrowed, zero-copy view and the way to pass any-length data; owned copies are explicit (`.toList()` via `bestie.lib.collections`, or `copyFrom` into an array)
* Conversions are explicit
* `ptr<T>` remains isolated in `memory.md`
* `list<T>` is a dynamic collection and lives in `bestie.lib.collections` — see `std-lib/collections.md`

This keeps the language small enough to reason about, while still making expressions like `5.toStr()`, `"x".isEmpty()`, `xs[-1]`, `xs[1..4]`, and `(0..10).contains(4)` feel natural and consistent.

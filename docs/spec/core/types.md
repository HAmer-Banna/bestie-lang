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
| `abs()` | same type | Absolute value |
| `isZero()` | `bool` | Fast zero test |
| `sign()` | `int8` | `-1`, `0`, or `1` |
| `countOnes()` | `int` | Population count |
| `leadingZeros()` | `int` | Bit width dependent |
| `trailingZeros()` | `int` | Bit width dependent |

Example:

```bestie
val n: int = -42
val s = n.toStr()
val a = n.abs()
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
| `isZero()` | `bool` | Fast zero test |
| `countOnes()` | `int` | Population count |
| `leadingZeros()` | `int` | Bit width dependent |
| `trailingZeros()` | `int` | Bit width dependent |

Example:

```bestie
val mask: uint32 = 0xFF00_FF00
val bits = mask.countOnes()

val raw: byte = 0x41
val text = raw.toStr()
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
| `abs()` | same type | Absolute value |
| `floor()` | same type | Rounds toward negative infinity |
| `ceil()` | same type | Rounds toward positive infinity |
| `round()` | same type | Rounds to nearest |
| `trunc()` | same type | Drops fractional part |
| `isNaN()` | `bool` | IEEE 754 NaN check |
| `isInfinite()` | `bool` | Positive or negative infinity |
| `isFinite()` | `bool` | True when neither NaN nor infinity |

Higher-level math such as `sqrt`, `pow`, trigonometry, and linear algebra remains in `bestie.lib.math`.

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

`!`, `&&`, and `||` are not boolean operators. `!` is reserved for `T ! E` and overflow-trap (`+!`). See `base.md` §15.

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
| `isAscii()` | `bool` | `0..=127` only |

Text-shaping behavior such as locale-aware casing, normalization, and segmentation belongs in `std-lib`.

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

`array<T>` is Bestie's built-in fixed-capacity contiguous collection.

It provides direct element storage, deterministic layout, and index-based access with **no hidden allocation** beyond what the declaration site requests.

`array<T>` is designed to be easy to use for the common case of holding a known number of values without forcing the programmer to reach for a dynamic collection.

### Construction Forms

| Form | Meaning |
| ---- | ------- |
| `array<T>[n]` | Fixed-capacity array of `n` elements, initially empty |
| `array<T>[] = {v1, v2, ...}` | Capacity and initial size inferred from the literal |
| `array<T>[n].fill(value)` | Fixed-capacity array, all `n` slots pre-filled with `value` |

Examples:

```bestie
val arr  : array<int>[5]                    // capacity 5, size 0 — fill with add()
val arr2 : array<int>[] = {1, 2, 3, 4, 5}  // capacity 5, size 5 — initialized from literal
val arr3 : array<int>[5] = array<int>[5].fill(0)  // capacity 5, size 5 — all zeros
```

`fill()` is the explicit escape hatch for pre-sized buffers and default-value grids, avoiding the need to write large literals. The value is copied into each slot — no implicit zero-filling ever occurs without it.

**Literal default type:** when a `{v, v, ...}` literal appears without a type annotation, the compiler infers `array<T>`. To obtain a different collection type from the same literal syntax, an explicit type annotation is required — see `base.md` §5.4.

### Operators

| Operation | Meaning |
| --------- | ------- |
| `arr[i]` | Element access — **panics** if index is out of bounds |
| `arr[-i]` | Access from the end — `arr[-1]` is the last element. Sugar for `arr[arr.size() - i]` |
| `arr[i] = v` | Element assignment when mutable — **panics** if out of bounds |
| `arr[-i] = v` | End-relative assignment when mutable — **panics** if out of bounds |
| `arr[lo..hi]` | Slice — a borrowed `slice<T>` view over `[lo, hi)`; **no copy**. See §6 |
| `for (x in arr)` | Iterate over all elements |

### Negative Indexing

A negative index counts from the end. `arr[-1]` is the last element, `arr[-2]` the second-to-last, and so on. It is pure syntax — `arr[-i]` lowers to `arr[arr.size() - i]`:

```bestie
val xs : array<int>[] = {10, 20, 30, 40, 50}

val last     = xs[-1]    // 50
val penult   = xs[-2]    // 40
xs[-1] = 99              // xs → {10, 20, 30, 40, 99}
```

Rules:

* `arr[-i]` is the **same operation** as `arr[size - i]` — one subtraction, then the normal access. There is no extra cost beyond positive indexing, and it is folded at compile time when `size` and the index are both statically known.
* Out-of-range negative indices **panic**, identical to positive out-of-range access (`arr[-6]` on a 5-element array panics).
* `0` and `-0` are the same index (the first element).

### Slicing

`arr[lo..hi]` produces a **`slice<T>`** — a borrowed, zero-copy view over a contiguous run of the array. It allocates nothing. The bound syntax matches `range<T>` (§7):

```bestie
val xs : array<int>[] = {10, 20, 30, 40, 50}

val a = xs[1..4]     // view over {20, 30, 40}   (half-open)
val b = xs[1..=3]    // view over {20, 30, 40}   (inclusive end)
val c = xs[2..]      // view over {30, 40, 50}
val d = xs[..2]      // view over {10, 20}
val e = xs[..]       // view over the whole array
val f = xs[-2..]     // view over the last two: {40, 50}
val g = xs[..-1]     // everything but the last: {10, 20, 30, 40}
```

A slice is governed by borrow rules — see §6 for the full semantics. Out-of-range bounds panic; `lo > hi` panics.

### Methods

| Method | Returns | Notes |
| ------ | ------- | ----- |
| `add(value: T)` | `void` | Appends element — **panics** if at capacity |
| `get(index: int)` | `T` | Explicit element access — **panics** if out of bounds |
| `size()` | `int` | Current number of elements |
| `capacity()` | `int` | Fixed maximum number of elements |
| `isEmpty()` | `bool` | True when no elements have been added |
| `isFull()` | `bool` | True when `size() == capacity()` |
| `toList()` | `list<T>` | New array-backed list containing all current elements |
| `toSet()` | `set<T>` | New hash set containing all current elements (deduplicates) |

Accessing a nonexistent index and adding beyond capacity are both **panics** — they represent violated invariants, not recoverable failures. See `exceptions.md`.

### Iteration

`array<T>` implements `Iterable<T>`. The `for/in` loop works with arrays naturally, with no import or conversion required:

```bestie
val scores : array<int>[] = {91, 84, 77}

for (s in scores) {
    print(s.toStr())
}
```

### Class Kind and Performance

`array<T>` has **value-class semantics**: no object header, no vtable, no identity, no inheritance.
It is a compiler-known built-in, not a user-declared class, but it follows the same rules as a `value class`:

* Elements are stored **contiguously in memory** — identical layout to a C array
* Element access at index `i` lowers to a single multiply-add-load sequence — one instruction, no indirection
* No bounds-check overhead on the happy path beyond the panic branch (which the CPU branch predictor learns immediately)
* No allocator metadata, no reference counting, no GC involvement

This is what is meant by "C-speed": `array<T>` access is not a method call with overhead — it is pointer arithmetic.

For classification purposes in the type system, `array<T>` behaves as a `value class`:

| Property | `array<T>` |
| -------- | ---------- |
| Object header | ❌ none |
| Vtable | ❌ none |
| Identity | ❌ none |
| Heap allocation | only when size requires it |
| Inheritance | ❌ not inheritable |
| Virtual dispatch | ❌ not applicable |

Compare to `list<T>` in `std-lib`, which is a `class` (heap-allocated, identity, owned backing buffer).

---

### Multi-Dimensional Arrays

Multi-dimensional arrays are expressed by nesting the size brackets.
`array<int>[rows][cols]` is shorthand for `array<array<int>[cols]>[rows]`.

When all dimensions are compile-time constants, the compiler lays the entire structure out as a **single flat contiguous block** (row-major order) — no pointer-to-pointer indirection, identical to a C 2D array:

```bestie
val grid  : array<int>[3][3]                           // 3×3, 9 contiguous ints
val mat   : array<float64>[][] = {{1.0, 2.0}, {3.0, 4.0}}  // inferred from 2D literal
val cube  : array<int>[4][4][4]                        // 4×4×4, 64 contiguous ints
```

Element access uses chained brackets, each level panicking independently on out-of-bounds:

```bestie
grid[1][2] = 42         // row 1, col 2 — single memory location, no indirection
val v = mat[0][1]       // 2.0
```

Rules:

* When all inner dimensions are fixed, the compiler guarantees flat row-major layout
* `array<T>[][cols]` is legal when the outer size is inferred from a literal
* Jagged arrays (variable-length rows) require `array<array<T>[]>` with explicit row sizes

---

### Immutability

`array<T>` does **not** use the `.immutable` builder modifier — it has no builder chain.
Two dedicated forms handle immutability instead:

**`const`** — compile-time literal arrays. Stored in `.rodata`. Zero runtime cost. Only valid with a literal right-hand side:

```bestie
const DAYS : array<str>[] = {"Mon", "Tue", "Wed", "Thu", "Fri"}
```

**`@immutable val`** — runtime arrays frozen after construction. The compiler rejects any mutation attempt as a compile-time error. Zero runtime overhead (no flag, no wrapper):

```bestie
@immutable val primes : array<int>[] = {2, 3, 5, 7, 11}
primes[0] = 1    // ❌ compile-time error: element mutation on @immutable binding
primes.add(13)   // ❌ compile-time error: add on @immutable binding
```

Full matrix:

| Declaration | Rebind? | Mutate elements? | Storage |
| ----------- | ------- | ---------------- | ------- |
| `var arr : array<int>[5]` | ✅ | ✅ | stack / heap |
| `val arr : array<int>[] = {1,2,3}` | ❌ | ✅ | stack / heap |
| `@immutable val arr : array<int>[] = {1,2,3}` | ❌ | ❌ | stack / heap |
| `const arr : array<int>[] = {1,2,3}` | ❌ | ❌ | `.rodata` |

`@immutable val` makes the array safe to share across threads with no locks — the compiler's guarantee that nothing mutates it is sufficient.

---

### Shared Interface with `list<T>`

`array<T>` shares its indexing syntax and common method names with `list<T>` in `std-lib`, so switching between them requires no interface relearning:

```bestie
arr[0]   // array element access
ls[0]    // list element access — same syntax

arr.size()  // 3
ls.size()   // 3 — identical method
```

The difference is that `array<T>` is **static**: its capacity is fixed at construction and never grows. This makes memory layout predictable and eliminates any risk of silent reallocation.

Example:

```bestie
val scores : array<int>[3]

scores.add(91)
scores.add(84)
scores.add(77)
// scores.add(60)  — panic: capacity exceeded

val first = scores[0]
val n     = scores.size()

for (s in scores) {
    print(s.toStr())
}
```

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
| `size()` | `int` | Number of elements in the view |
| `isEmpty()` | `bool` | True when `size() == 0` |
| `toArray()` | `array<T>` | New owned array (O(n) copy — explicit) |
| `toList()` | `list<T>` | New owned list (O(n) copy — explicit) |

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

### Immutable Sources

Slicing immutable storage is always safe and never requires a copy:

* A slice over a `str`, a `const` array, or an `@immutable val` array is inherently read-only.
* Because the source cannot mutate in place, the "no mutation while borrowed" rule is satisfied for free — the only remaining constraint is lifetime (the slice must not outlive the source).

### Class Kind and Performance

`slice<T>` has **value-class semantics** — no object header, no vtable, no identity, no heap. It is a fat pointer (base + length) passed in registers. This is what keeps slicing zero-cost: taking a slice is two register writes, and indexing through it is the same single load as indexing the underlying array.

> What slicing does **not** do: it never copies, never allocates, and never owns. To obtain an owned, escapable copy, call `.toArray()` or `.toList()` explicitly. Only contiguous sources produce a `slice<T>`; non-contiguous collections are covered in `std-lib/collections.md`.

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
| `size()` | `int` | Number of values in the range |
| `contains(value: T)` | `bool` | Membership test |
| `mid()` | `T` | Midpoint when meaningful for `T` |
| `isEmpty()` | `bool` | Empty or reversed range |

Example:

```bestie
val r: range<int> = 0..100

val n = r.size()
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

This syntax is sealed with the rest of core. The named type `option<T>`, its constructors (`Present` / `Not_Present`), and any helper methods live in **`bestie.lib.utilities`** — part of the language, not part of core. See `std-lib/util.md`. `int ?` and `option<int>` are the same representation; the names can evolve with the library, the `?` spelling cannot.

**Core surface (no import):**

* Signatures, parameters, and fields spelled `T ?`
* On `fun f(): T ?`: `return value` is present; a bare `return` is absent
* `if (val x = …)` binds only when present (`fp.md` §3)
* FFI: C `NULL` maps to `ptr<T> ?` (`foreign.md`)

**Std-lib surface (`import bestie.lib.utilities`):**

* The name `option<T>`
* Matching `option.Present` / `option.Not_Present`
* Generics that read better as `list<option<User>>` (equivalently `list<User ?>` without import)

Default parameters (`x: int = 0`) are a different tool: the caller may omit the argument and the compiler fills a compile-time constant. `x: int ?` means the value may be absent at runtime. Both are valid; they are not interchangeable.

`?` binds to the immediately preceding type: `list<str ?>` is a list of optional strings; `list<str> ?` is an optional list.

---

## 8.4 Error unions — `T ! E`

`T ! E` is **core syntax**. `E` is an error set (see `exceptions.md`). This is recoverable failure, not absence.

```bestie
fun parse(s: str): int ! ParseError
```

The named type `result<T, E>`, its constructors (`Ok` / `Err`), and any helper methods live in **`bestie.lib.utilities`**. `int ! ParseError` and `result<int, ParseError>` are the same representation.

**Core surface (no import):**

* Signatures spelled `T ! E`
* `try` / `catch` at the call site
* Type arguments that are “a value or an error” — `Channel<int ! WorkError>`

**Std-lib surface (`import bestie.lib.utilities`):**

* The name `result<T, E>`
* Matching `result.Ok` / `result.Err`

Function signatures should use `T ! E`. Do not invent a second error API.

A function may not return both `?` and `!` on the same slot (`T ? ! E` is rejected). Absence and failure are different: use `T ?` for “no value”, `T ! E` for “this failed”.

---

## 9. Summary

Bestie's core type surface is intentionally compact:

* Primitive numeric values behave like zero-cost value classes
* `bool`, `char`, and `str` expose explicit core operations only
* `tuple`, `array<T>`, `slice<T>`, `range<T>`, `T ?`, and `T ! E` are first-class core forms. Named `option<T>` / `result<T, E>` live in `bestie.lib.utilities`
* Indexing is uniform: `[i]` panics out of bounds, `[-i]` counts from the end, `[lo..hi]` slices — the same on `array<T>`, `str`, and (in std-lib) array-backed `list<T>`
* `slice<T>` is a borrowed, zero-copy view; owned copies are explicit (`.toArray()` / `.toList()`)
* Conversions are explicit
* `ptr<T>` remains isolated in `memory.md`
* `list<T>` is a dynamic collection and lives in `bestie.lib.collections` — see `std-lib/collections.md`

This keeps the language small enough to reason about, while still making expressions like `5.toStr()`, `"x".isEmpty()`, `xs[-1]`, `xs[1..4]`, and `(0..10).contains(4)` feel natural and consistent.

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
* `@trusted` may suppress runtime checks, with the same responsibility model as other unsafe escape hatches

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

Higher-level math such as `sqrt`, `pow`, trigonometry, and linear algebra remains in `std-lib.math`.

---

### 2.5 `bool`

`bool` is a primitive logical value with exactly two states:

* `true`
* `false`

#### Operators

`bool` supports:

* Logical negation: `!`
* Logical conjunction: `&&`
* Logical disjunction: `||`
* Equality and inequality: `==`, `!=`

#### Methods

| Method | Returns | Notes |
| ------ | ------- | ----- |
| `toStr()` | `str` | `"true"` or `"false"` |
| `toInt()` | `int` | `1` or `0` |
| `not()` | `bool` | Method form of logical negation |

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

### Methods

| Method | Returns | Notes |
| ------ | ------- | ----- |
| `toStr()` | `str` | Identity conversion |
| `isEmpty()` | `bool` | Zero bytes |
| `byteSize()` | `int` | UTF-8 byte length |
| `char(index: int)` | `char` | Unicode scalar at codepoint index |
| `chars()` | iterator over `char` | Iterates over Unicode scalars |

Rules:

* `s[i]` is raw byte access, not character access
* `s.char(i)` is the explicit Unicode-aware path
* Bestie intentionally does not overload a single ambiguous `length()` meaning in core

Example:

```bestie
val s: str = "Bestie"

val firstByte: byte = s[0]
val firstChar: char = s.char(0)
val empty: bool = s.isEmpty()
```

Searching, splitting, normalization, case conversion, and rich text processing belong in `std-lib`.

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

**Literal default type:** when a `{v, v, ...}` literal appears without a type annotation, the compiler infers `array<T>`. To obtain a different collection type from the same literal syntax, an explicit type annotation is required — see `lang.md` §5.4.

### Operators

| Operation | Meaning |
| --------- | ------- |
| `arr[i]` | Element access — **panics** if index is out of bounds |
| `arr[i] = v` | Element assignment when mutable — **panics** if out of bounds |
| `for (x in arr)` | Iterate over all elements |

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

## 6. `range<T>`

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

## 7. Derived Core Type Forms

Bestie also has two important **type forms** built on top of the core type system.

### 7.1 Newtypes

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

### 7.2 Range-Constrained Types

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

## 8. Summary

Bestie's core type surface is intentionally compact:

* Primitive numeric values behave like zero-cost value classes
* `bool`, `char`, and `str` expose explicit core operations only
* `tuple`, `array<T>`, and `range<T>` are first-class built-in value forms
* Conversions are explicit
* `ptr<T>` remains isolated in `memory.md`
* `list<T>` is a dynamic collection and lives in `bestie.lib.collections` — see `std-lib/collections.md`

This keeps the language small enough to reason about, while still making expressions like `5.toStr()`, `"x".isEmpty()`, and `(0..10).contains(4)` feel natural and consistent.

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

## 5. `list<T>`

`list<T>` is the only collection built into the core language.

It provides the language's array semantics and participates directly in memory, loop, and ownership rules.

### Construction Forms

| Form | Meaning |
| ---- | ------- |
| `list<T>.build()` | Default builder |
| `list<T>.array.build()` | Explicit array-backed list |
| `list<T>.linked.build()` | Explicit linked variation |
| `{1, 2, 3}` | List literal when target type is `list<T>` |
| `list<int>[10]` | Sized array form |
| `list<int>[2][3]` | Multi-dimensional sized array form |

### Operators

| Operation | Meaning |
| --------- | ------- |
| `xs[i]` | Element access |
| `xs[i] = v` | Element assignment when mutable |
| `for (x in xs)` | Iteration |

### Core Methods

| Method | Returns | Notes |
| ------ | ------- | ----- |
| `build()` | `list<T>` | Finalizes builder chain |
| `add(value: T)` | `list<T>` | Appends element |
| `insert(index: int, value: T)` | `list<T>` | Inserts at position |
| `get(index: int)` | `T` | Explicit element access |
| `remove(index: int)` | `T` | Removes and returns element |
| `indexOf(value: T)` | `int ?` | Returns index or partial result |
| `size()` | `int` | Number of elements |
| `isEmpty()` | `bool` | True when no elements exist |

### Modifiers and Variations

The core list model also supports variation and behavior selection through the builder chain:

* `array`
* `linked`
* `immutable`
* `copyOnWrite`
* `concurrent`

Example:

```bestie
val xs = list<int>.array.immutable.build()
var ys = list<str>.linked.build()

ys.add("a")
ys.insert(0, "z")
```

All deeper list memory semantics are defined in `memory.md`.

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
* `tuple`, `list<T>`, and `range<T>` are first-class built-in value forms
* Conversions are explicit
* `ptr<T>` remains isolated in `memory.md`

This keeps the language small enough to reason about, while still making expressions like `5.toStr()`, `"x".isEmpty()`, and `(0..10).contains(4)` feel natural and consistent.

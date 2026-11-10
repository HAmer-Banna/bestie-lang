# Bestie Math Module

This document defines the **core mathematical abstractions** provided by Bestie and the separation between **core numeric types** and **standard-library mathematical utilities**.

Math in Bestie is:

* Deterministic
* Runtime-independent
* Explicit
* Compile-time friendly

Bestie deliberately avoids implicit numeric magic or operator-heavy abstractions that obscure cost or behavior.

---

## 1. Scope and Layering

Bestie math is divided into two layers:

1. **Core Math (language level)**
2. **Math Std-Lib (`bestie.lib.math`)**

Only primitives that must be understood by the compiler live in core. Everything else is library code.

---

## 2. Core Numeric Types

Core numeric primitives match the **Primitive Types** section of the [core language spec](../core/lang.md). In short:

* **Signed integers:** `int8`, `int16`, `int32`, `int64`, `int128`, and pointer-sized `int`
* **Unsigned integers:** `uint8`, `uint16`, `uint32`, `uint64`, `uint128`, and pointer-sized `uint` (`byte` is an alias for `uint8`)
* **Floating-point:** `float16`, `float32`, `float64`, and target-default `float` (IEEE 754)

### 2.1 Design Rules

* All numeric types have **well-defined size and overflow behavior**
* No implicit widening or narrowing
* Conversions are **explicit**
* No boxed numbers or hidden heap allocation

```bestie
val x : int = 10
val y : int64 = x as int64   // explicit
```

---

## 3. Arithmetic Semantics

* Arithmetic is **strict and predictable**
* Signed integer overflow is **defined** (wrap in release, trap in debug; see core spec), not undefined behavior
* Overflow on constant expressions may be caught at compile time where provable

```bestie
val x : int = 2_000_000_000
val y = x + x   // compile-time error or explicit overflow handling
```

---

## 4. `matrix<T>`

Core has no matrix type. Compile-time grids that are **storage only** stay as nested `array<T>` (`array<int>[3][3]` in `core/types.md`). Algebra is not a language primitive.

`matrix<T>` lives in `bestie.lib.math`. It is a **`class`**, lowercase like `list` and `set`: a foundational library type, not a nominal domain object.

### 4.1 Class kind

| Type | Kind | Reasoning |
| ---- | ---- | --------- |
| `matrix<T>` | `class` | Owns a heap buffer whose size is `rows * cols` at runtime; has identity; supports in-place mutation |

It is not a `value class`: those cannot hold `own` fields, are copy-by-value, and cannot own a runtime-sized buffer. Copying a matrix on assignment would silently duplicate storage and hide allocation.

It is not a collection. Dimensions are fixed at construction. There is no `.add`, no builder growth, and no `list<T>.matrix` view.

`T` is a core numeric primitive (`int*`, `uint*`, `float16`, `float32`, `float64`, `float`).

### 4.2 Layout and construction

Storage is a single contiguous **row-major** buffer. Index `(r, c)` is `r * cols + c`. No pointer-to-pointer rows.

```bestie
import bestie.lib.math

val a = matrix<float64>.zeros(3, 3)
val b = matrix<float64>.identity(3)
val c = matrix<float64>.new(2, 2, {1.0, 2.0, 3.0, 4.0})  // row-major, length == rows * cols
```

Rules:

* `rows` and `cols` must be `> 0` — a zero dimension is a panic (violated invariant)
* `.of` panics if the literal length is not `rows * cols`
* Out-of-range `get` / `set` is a panic, same as `array<T>`

### 4.3 Operations

Higher-level math uses **named methods**, not `*` / `+` overloads (see §5).

```bestie
import bestie.lib.math

val a = matrix<float64>.identity(3)
val b = matrix<float64>.zeros(3, 3)
b.set(0, 1, 2.0)

val c = a.mul(b)          // new matrix; allocation is at this call
val d = a.add(b)
val t = a.transpose()

a.mulInPlace(b)           // mutates `a`; no extra allocation
```

Shape mismatches on `mul` / `add` / `mulInPlace` are `! DimensionError` — the sizes are data, not a broken invariant of a single matrix.

```bestie
errors DimensionError {
    IncompatibleShape
}
```

Indexing:

```bestie
val x = a.get(0, 1)
a.set(0, 1, 3.5)
val rows = a.rows()
val cols = a.cols()
```

### 4.4 What `matrix<T>` is not

* Not a replacement for `array<T>[n][m]` — use arrays when the size is compile-time and you only need storage
* Not a tensor library — higher rank stays out of this module until a concrete need exists
* Not a collection — it does not implement the collections builder API

SIMD or hardware acceleration may apply inside the implementation; it is not exposed as a second type.

---

## 4a. Numeric Primitive Extensions

Core keeps only **operators** and **explicit conversions** on numeric primitives (`core/types.md` §2.7). Everything else in the numeric surface lives here, as **extension functions** — so it reads as a method at the call site while remaining freely extensible:

```bestie
import bestie.lib.math

val n: int = -42
val a = n.abs()
val bits = mask.countOnes()
val f = (2.7).floor()
```

### 4a.1 Integer extensions

Available on every signed and unsigned integer type (`int8`…`int64`, `int`, `uint8`…`uint64`, `uint`, `byte`) unless noted.

| Extension | Returns | Notes |
| --------- | ------- | ----- |
| `abs()` | same type | Signed types only. `int8.MIN.abs()` overflows — traps in debug, wraps in release (`core/lang.md` §22.2); use `absChecked()` for a `! OverflowError`. |
| `absChecked()` | same type `! OverflowError` | Signed types only |
| `sign()` | `int8` | `-1`, `0`, or `1`. Signed types only |
| `isZero()` | `bool` | Fast zero test |
| `countOnes()` | `int` | Population count |
| `countZeros()` | `int` | Width minus `countOnes()` |
| `leadingZeros()` | `int` | Bit-width dependent |
| `trailingZeros()` | `int` | Bit-width dependent |
| `rotateLeft(n: int)` | same type | Bitwise rotation |
| `rotateRight(n: int)` | same type | Bitwise rotation |
| `isPowerOfTwo()` | `bool` | Unsigned types only |
| `nextPowerOfTwo()` | same type | Unsigned types only; saturates at `T.MAX` |

### 4a.2 Floating-point extensions

Available on `float16`, `float32`, `float64`, and `float`.

| Extension | Returns | Notes |
| --------- | ------- | ----- |
| `abs()` | same type | Clears the sign bit; total |
| `floor()` | same type | Toward negative infinity |
| `ceil()` | same type | Toward positive infinity |
| `round()` | same type | Nearest, halfway away from zero |
| `trunc()` | same type | Drops the fractional part |
| `fract()` | same type | `x - x.trunc()` |
| `signum()` | same type | `-1.0`, `0.0`, `1.0`, or NaN |
| `copySign(to: T)` | same type | IEEE 754 `copysign` |

`isNaN()`, `isInfinite()`, and `isFinite()` stay in **core** (`core/types.md` §2.4): `core/constants.md` makes `.isNaN()` the mandated way to test `T.NAN`, so core must supply it.

### 4a.3 Cost

Extension functions are statically resolved and monomorphized (`core/fp.md` §11.2). `n.abs()` compiles to the same instruction sequence it would as a built-in method — there is no dispatch, no wrapper, and no allocation. The only difference is which layer may change it.

### 4a.4 Why these are here and not in core

This set is the part of the numeric surface that keeps growing. `rotateLeft`, `nextPowerOfTwo`, and `copySign` in the tables above are all additions core could not have made without a language release. Not knowing that Bestie has `countOnes` teaches a programmer nothing wrong about the language; not knowing it has `data class` does.

Core keeps the operators, because those must mean the same thing in Bestie 1 and Bestie 10.

---

## 4b. Big Numbers

Numbers beyond the core types come in two shapes, and Bestie gives each a different design:

| Type | Kind | Size known | Operators | Main users |
| ---- | ---- | ---------- | --------- | ---------- |
| `wideInt<BITS>` / `wideUInt<BITS>` | `value class`, inline | at compile time | ✅ | cryptography, hashing, wide fixed-point |
| `BigInt` | `class`, heap | at run time | ❌ — named methods | compilers, parsers, number theory |
| `decimal64` / `decimal128` | `value class`, inline | at compile time | ✅ | money and finance |

**Why the heap type has no operators.** Bestie has no garbage collector and no implicit cleanup (`core/memory.md` §7.4). If `BigInt` had `+`, then `a * b + c` would allocate a temporary for `a * b` that nothing could free — and an operator that allocates breaks the rule that nothing allocates behind syntax that looks free (`vision.md` Rule 3). Types whose size is fixed at compile time have no such problem: they are values, and `+` on them is as free as on `int64`.

### 4b.1 `wideInt<BITS>` and `wideUInt<BITS>` — Fixed-Width Big Integers

```bestie
import bestie.lib.math

const P = wideUInt<256>.fromHex("FFFFFFFF_FFFFFFFF_FFFFFFFF_FFFFFFFF_FFFFFFFF_FFFFFFFF_FFFFFFFE_FFFFFC2F")

fun mulMod(a: wideUInt<256>, b: wideUInt<256>): wideUInt<256> {
    val wide = a.toWideUInt<512>() * b.toWideUInt<512>()      // exact: 256 × 256 fits 512
    return (wide % P.toWideUInt<512>()).toWideUInt<256>() ?: panic("fits after % P")
}
```

* **Representation.** A `value class` holding `array<uint64>[BITS / 64]`, least significant limb first: `sizeOf` is `BITS / 8`, alignment 8. No heap, no header, copyable (`core/memory.md` §6.1). `BITS` is a compile-time value parameter (`core/lang.md` §7.2): a multiple of 64, from 128 to 65536. `wideInt` is two's complement.
* **Operators.** `+` `-` `*` `/` `%`, unary `-` on `wideInt`, `==`, and the ordering operators, through the operator protocols (`utils.md` §6) — statically dispatched and inlined. Overflow follows the core integer rules (§3). Explicit forms: `addWrapping(b)`, `subWrapping(b)`, `mulWrapping(b)` for arithmetic modulo 2^BITS, and `addChecked(b)`, `subChecked(b)`, `mulChecked(b)` returning `! OverflowError`.
* **Bits.** Bitwise operators are not overloadable (`core/lang.md` §15.3), so bit operations are methods: `bitAnd(b)`, `bitOr(b)`, `bitXor(b)`, `bitNot()`, `shiftLeft(n)`, `shiftRight(n)` (arithmetic on `wideInt`), and `bit(i): bool`.
* **Constant time.** Addition, subtraction, multiplication, the bitwise methods, `==`, and ordering run in time that depends only on `BITS`, never on the values — no branch and no memory access depends on a value. That is the property cryptographic code needs. Division, remainder, and shifts by a value-dependent amount are not constant-time, and say so in their documentation.
* **Conversions.** `toWideUInt<M>()` / `toWideInt<M>()` widen exactly when `M >= BITS` and narrow to `T ?` otherwise (absent when the value does not fit). From and to core integers: `wideUInt<256>.from(x)` for any core integer `x`, and `toUInt64()`, `toUInt128()` and their signed forms, which narrow to `T ?`.
* **Text.** `fromHex(s)` and `fromDecimal(s)` return `T ! ParseError` (`strings.md`). In a `const` initializer they are evaluated at compile time: invalid text is a compile-time error, so the constant's type is `T` and no `try` is written. `toHex()` and `toStr()` give text.
* `wideUInt<128>` has the layout of core's `uint128` and converts to it exactly. Use `uint128` for 128-bit work: it lives in registers and the compiler knows it (`core/types.md` §2.2); `wideUInt` is for widths core does not have.

### 4b.2 `BigInt` — Arbitrary Precision

`BigInt` is an integer whose size is only known at run time: a compiler folding constants of any size, a parser reading a JSON number of any length, number-theory code. It is a **`class`** that owns a heap buffer of limbs, and it is written the way fast arbitrary-precision code is written in C (GMP): **every operation writes into a result you name**, so each allocation is visible and buffers are reused across a loop.

```bestie
import bestie.lib.math

val own a = try BigInt.parse("123456789012345678901234567890")
val own b = BigInt.from(1_000_003)
val own r = BigInt.new()                       // zero
defer a.free()
defer b.free()
defer r.free()

r.setProduct(a.address(), b.address())         // r = a * b
r.setSum(r.address(), b.address())             // r = r + b — the result may alias an operand
log(r.toStr())
```

| Method on the result `r` | Effect |
| ------------------------ | ------ |
| `setSum(a, b)` · `setDifference(a, b)` · `setProduct(a, b)` | `r = a + b` · `a - b` · `a * b` |
| `setQuotient(a, b)` · `setRemainder(a, b)` | Truncating division and its remainder; **panics** when `b` is zero |
| `setPower(a, e: uint)` | `r = a ^ e` |
| `setNegation(a)` · `copyFrom(a)` | `r = -a` · `r = a` |
| `reserve(bits: uint)` | Grow the buffer once, so a loop that stays within `bits` allocates nothing |

Rules:

* Operands are `ptr<const BigInt>`, as for any shared `class` (`core/memory.md` §6.3). The result may be one of its own operands: `r.setSum(r.address(), b.address())` is defined.
* A `set*` method grows `r`'s buffer when the result needs more limbs. That is the one allocation, and it is in the method's name and documentation; running out of memory panics, as for collections. A result never shrinks its buffer unless `shrink()` is called.
* `BigInt.new()`, `BigInt.from(x)` for any core integer, and `BigInt.parse(s): own BigInt ! ParseError` construct. `toStr()` returns an `own str`. `compareTo(other)` and `equal(other)` make `BigInt` `Comparable` and `Equable` (`utils.md`), so ordering and `==` work, because comparing allocates nothing.
* `BigInt` does **not** implement `Addable` or the other arithmetic protocols, so `a + b` on it is a compile-time error.
* It allocates from `heap` (`std-lib/allocators.md` §4a.3). An allocator-aware form belongs with roadmap item P2.
* `free()` releases the buffer; ownership rules are those of any `own` class (`core/memory.md` §7).

### 4b.3 `decimal64` and `decimal128` — Exact Decimal Arithmetic

Binary floats cannot represent `0.1`, so `0.1 + 0.2 != 0.3` — wrong for money. `decimal64` and `decimal128` are the IEEE 754-2008 decimal formats: every decimal number with up to 16 (`decimal64`) or 34 (`decimal128`) significant digits is stored exactly.

```bestie
import bestie.lib.math

const PRICE    = decimal128.parse("19.99")
const TAX_RATE = decimal128.parse("0.0825")

val total = PRICE * decimal128.from(3)         // exactly 59.97
val tax   = (total * TAX_RATE).roundTo(places: 2)
```

* **Representation.** `value class`es of 8 and 16 bytes, aligned to their size, in the binary integer decimal (BID) encoding — the encoding C23's `_Decimal64` / `_Decimal128` use on x86-64 and AArch64, so they cross the C boundary unchanged.
* **Operators.** `+` `-` `*` `/`, unary `-`, `==`, and ordering, through the operator protocols. Results are correctly rounded to the format's precision, half to even by default. `+`, `-`, and `*` are exact whenever the result fits the precision, which for currency amounts in `decimal128` is always.
* **Rounding is named.** `roundTo(places: int, mode: RoundingMode = RoundingMode.HalfEven)` with `HalfEven`, `HalfUp`, `Down`, `Up`, `Floor`, `Ceiling`. Division that cannot be exact rounds at the precision limit; code that needs cents rounds explicitly.
* **IEEE behavior.** Overflow gives infinity, invalid operations give NaN; `isNaN()`, `isInfinite()`, and `isFinite()` work as on binary floats. `decimal128.MAX`, `MIN`, `INFINITY`, and `NAN` exist.
* **Text and construction.** `parse(s): T ! ParseError` reads decimal text exactly. In a `const` initializer it is evaluated at compile time, as `fromHex` is (§4b.1), so the constant is a plain `T`. `from(x)` takes any core integer exactly. There is deliberately no conversion from a binary float literal: `0.1` as a `float64` is already not `0.1`. `toStr()` prints the exact digits.
* **Cost.** No mainstream CPU computes decimal arithmetic in hardware (IBM POWER aside), so it is implemented in Bestie, without allocation. It is several times slower than binary floating point and gives the same result on every target.

### 4b.4 Arbitrary-Precision Floats

There is no `BigFloat`. Arbitrary-precision binary floating point (MPFR-style) is rarely needed, large to implement, and needs a precision chosen per operation — a third-party package's job (§7). Exact decimal arithmetic, which is what most code asking for a big float wants, is §4b.3.

---

## 5. Operators vs Functions

Core math favors **explicit functions** over overloaded operators.

Operators are limited to:

* Basic arithmetic (`+`, `-`, `*`, `/`)
* Comparison

Higher-level math uses named functions:

```bestie
import bestie.lib.math

val x = sqrt(16)
val y = pow(2, 8)
```

---

## 6. Determinism and Performance

Math operations in Bestie:

* Have no hidden allocations
* Are deterministic across platforms
* Can be evaluated at compile time when inputs are constant

```bestie
const x: int = pow(2, 10)  // resolved at compile time
```

---

## 7. What Math Does Not Include

Core and std-lib math intentionally exclude:

* Symbolic math
* Arbitrary-precision binary floats (§4b.4)
* Implicit vectorization
* Runtime-dependent optimizations
* Lazy evaluation

These may be built as external libraries.

---

## 8. Summary

Bestie math design:

* Keeps core minimal and predictable
* Gives algebra its own type (`matrix<T>`), separate from `array<T>` storage
* Provides big numbers in the shape that fits each use: fixed-width `wideInt<BITS>` / `wideUInt<BITS>` and exact `decimal64` / `decimal128` as values with operators, and heap `BigInt` with named, allocation-visible methods
* Pushes advanced math into explicit libraries
* Favors clarity over cleverness

> Math in Bestie is **explicit by default and powerful by choice**.

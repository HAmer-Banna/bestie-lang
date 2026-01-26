Below is a **revised and complete `math.md`** suitable for **Bestie core / std-lib documentation**.
It is intentionally **substantial**, explicit, and aligned with Bestie’s principles (performance-first, no magic, compile-time clarity).

---

# Bestie Standard Library — Math (`math.md`)

This document defines the **mathematics facilities** provided by Bestie’s standard library.

Math in Bestie is designed to be:

* **Deterministic**
* **Precise**
* **Explicit**
* **Allocation-aware**
* **Zero-surprise**

Bestie does not blur the line between **numeric computation** and **numeric abstraction**.
Each numeric type exists for a clear, non-overlapping purpose.

---

## 1. Design Philosophy

Bestie math follows these rules:

1. **Primitive math is fast**
2. **Precision is explicit**
3. **Overflow is never silent**
4. **Abstraction never hides cost**
5. **Numeric intent must be visible in code**

There is no implicit promotion, no hidden heap allocation, and no automatic precision escalation.

If you want more precision — you ask for it.

---

## 2. Primitive Numeric Types (Core)

These types are part of the **language core** and require no imports.

### 2.1 Integers

| Type   | Size   | Signed |
| ------ | ------ | ------ |
| int8   | 8-bit  | Yes    |
| int16  | 16-bit | Yes    |
| int32  | 32-bit | Yes    |
| int64  | 64-bit | Yes    |
| uint8  | 8-bit  | No     |
| uint16 | 16-bit | No     |
| uint32 | 32-bit | No     |
| uint64 | 64-bit | No     |

Rules:

* Overflow is **checked by default**
* Overflow behavior is deterministic
* Unsafe wraparound is not allowed in core math

Example:

```bestie
val a: int32 = 2_000_000_000
val b = a * 2   // ❌ compile-time or runtime error (overflow)
```

Explicit overflow handling is provided via std APIs.

---

### 2.2 Floating Point

| Type    | Standard |
| ------- | -------- |
| float32 | IEEE 754 |
| float64 | IEEE 754 |

Rules:

* No implicit int → float widening
* NaN and Infinity are preserved
* Floating-point math follows platform IEEE guarantees

Example:

```bestie
val x: float64 = 0.1
val y = x + 0.2
```

Floating-point math is **fast**, not magical.

---

### 2.3 Character and Boolean

* `char` represents a Unicode scalar value
* `bool` is a true boolean, not an integer alias

They are **not** numeric types and cannot be used in arithmetic.

---

## 3. Math Utility Functions

Provided via `std.math`.

### 3.1 Basic Operations

```bestie
abs(x)
min(a, b)
max(a, b)
clamp(value, min, max)
```

These are:

* Inlineable
* Type-safe
* Overload-resolved at compile time

---

### 3.2 Trigonometry

```bestie
sin(x)
cos(x)
tan(x)
asin(x)
acos(x)
atan(x)
atan2(y, x)
```

Rules:

* Operate on `float32` / `float64`
* No implicit degree/radian conversion
* Radians only

---

### 3.3 Exponential & Logarithmic

```bestie
sqrt(x)
pow(base, exp)
exp(x)
log(x)
log10(x)
```

Precision is determined solely by the input type.

---

## 4. BigInteger

### 4.1 Purpose

`BigInteger` exists for **arbitrary-precision integer arithmetic**, where overflow is unacceptable.

Typical use cases:

* Cryptography
* Financial ledgers
* Scientific computation
* Large counters

---

### 4.2 Type Definition

```bestie
class BigInteger
```

Properties:

* Arbitrary precision
* Heap-allocated
* Immutable by default
* Thread-safe

---

### 4.3 Creation

```bestie
val a = BigInteger.of("12345678901234567890")
val b = BigInteger.of(42)
```

No implicit conversion from primitive integers.

---

### 4.4 Operations

```bestie
a + b
a - b
a * b
a / b
a % b
a.pow(10)
```

Rules:

* All operations allocate
* Cost is explicit
* No silent truncation

---

## 5. BigDecimal

### 5.1 Purpose

`BigDecimal` provides **arbitrary-precision decimal arithmetic** with predictable rounding.

It exists to solve problems that floating-point **cannot** safely represent.

Use cases:

* Finance
* Accounting
* Pricing systems
* Exact decimal math

---

### 5.2 Type Definition

```bestie
class BigDecimal
```

Properties:

* Arbitrary precision
* Base-10 representation
* Immutable
* Thread-safe

---

### 5.3 Creation

```bestie
val x = BigDecimal.of("0.1")
val y = BigDecimal.of(10)
```

Strings are preferred to avoid representation ambiguity.

---

### 5.4 Operations

```bestie
x + y
x - y
x * y
x / y
```

Division rules:

* Requires explicit rounding mode
* No implicit rounding

Example:

```bestie
x.divide(y, Rounding.HalfUp)
```

---

## 6. Rounding & Precision Control

### 6.1 Rounding Modes

Provided as an enum:

```bestie
enum class Rounding {
    Up,
    Down,
    HalfUp,
    HalfDown,
    HalfEven
}
```

No default rounding is assumed.

---

## 7. Numeric Protocols

### 7.1 Comparable

All numeric types implement:

```bestie
protocol Comparable {
    fun compare(other: Self): int
}
```

Comparison is:

* Total
* Deterministic
* Compile-time resolved where possible

---

### 7.2 Hashable

All numeric types implement:

```bestie
protocol Hashable {
    fun hash(): int
}
```

Hashing is stable and platform-independent.

---

## 8. Interoperability Rules

### 8.1 No Implicit Promotion

The following is forbidden:

```bestie
val x: BigInteger = 10    // ❌
```

Must be explicit:

```bestie
val x = BigInteger.of(10)
```

---

### 8.2 No Implicit Precision Upgrade

Bestie will **never** silently upgrade:

* int → BigInteger
* float → BigDecimal

Precision is always a deliberate choice.

---

## 9. Performance Guarantees

* Primitive math is zero-cost abstraction
* BigInteger and BigDecimal costs are explicit
* No hidden allocations
* No runtime type checks

If you see `Big*`, you know you are paying for precision.

---

## 10. What Bestie Does NOT Include

Bestie intentionally excludes:

* Complex numbers (can be built via value classes)
* Symbolic math
* Operator overloading with hidden allocation
* Implicit numeric coercion

These can be provided by **libraries**, not core.

---

## 11. Summary

Bestie math is:

* **Fast for primitives**
* **Exact when requested**
* **Explicit in cost**
* **Safe by construction**
* **Free of numeric magic**

You choose:

* Speed or precision
* Stack or heap
* Fixed or arbitrary size

And the compiler enforces your choice.

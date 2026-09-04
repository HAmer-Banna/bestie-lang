# bestie.lib.strings — String Parsing and Text Operations

This document defines the **Bestie Standard String Library (`bestie.lib.strings`)**.

Core `str` (see `core/types.md` §3) is intentionally minimal — it exposes only raw byte access, codepoint access, length, and iteration. Everything higher-level — **parsing**, **substrings**, **search**, **splitting**, **trimming**, and **case conversion** — lives here.

`bestie.lib.strings` is delivered entirely as **extension functions** on `str` (and a few on `list<str>`). This means:

* Core stays small; `str` gains no new fields or built-in methods.
* The operations still read as methods (`s.toInt()`, `s.trim()`) because extension functions are called with method syntax (`fp.md` §11).
* Every function is statically resolved and monomorphized — no dynamic dispatch, no runtime cost beyond the work itself.

```bestie
import bestie.lib.strings
```

---

## 1. Design Principles

1. **`str` is immutable** — every operation returns a **new** `str`; nothing mutates in place.
2. **Codepoint-aware by default** — indices in this module are **Unicode scalar (codepoint)** indices, not byte offsets. Byte-level access stays in core (`s[i]`, `s[lo..hi]`).
3. **Parsing is explicit and fallible** — string→value conversions return `T ! ParseError`; there is no silent default.
4. **No implicit locale** — case conversion uses Unicode default case mapping. Locale-sensitive behavior is out of scope for this module.
5. **No hidden allocation surprises** — allocation happens only where a new `str` or `list<str>` is produced, and that is always visible at the call site.

---

## 2. Parsing

String→value parsing is provided as fallible extension functions, reusing the shared `ParseError` (see `std-lib/format.md` §10).

```bestie
fun str.toInt():     int     ! ParseError
fun str.toInt8():    int8    ! ParseError
fun str.toInt16():   int16   ! ParseError
fun str.toInt32():   int32   ! ParseError
fun str.toInt64():   int64   ! ParseError
fun str.toUInt():    uint    ! ParseError
fun str.toUInt64():  uint64  ! ParseError
fun str.toFloat32(): float32 ! ParseError
fun str.toFloat64(): float64 ! ParseError
fun str.toBool():    bool    ! ParseError   // "true" / "false"
```

Rules:

* Surrounding whitespace is **not** trimmed implicitly — call `.trim()` first if needed.
* Out-of-range values (e.g. an integer too large for the target type) yield `ParseError.Overflow`.
* Malformed input yields `ParseError.SyntaxError`.
* A radix-aware variant is available for integers: `fun str.toIntRadix(radix: int): int ! ParseError` (radix `2..=36`).

```bestie
import bestie.lib.strings

val port = "8080".toInt() catch |e| { 0 }
val mask = "ff".toIntRadix(16) catch |e| { 0 }   // 255
```

These mirror the total, infallible numeric `n.toStr()` from core — the inverse direction is fallible, so it is explicit here rather than in core.

---

## 3. Substrings and Slicing

`substring` is **codepoint-aware** and returns an owned `str`. Bounds are half-open, matching the `..` convention.

```bestie
fun str.substring(start: int, end: int): str   // [start, end) in codepoints
fun str.substringFrom(start: int): str          // [start, end-of-string)
fun str.take(n: int): str                        // first n codepoints
fun str.drop(n: int): str                        // all but the first n codepoints
```

Rules:

* Indices are **Unicode scalar** positions, so a substring never splits a multi-byte codepoint.
* Out-of-range indices (or `start > end`) **panic** — a violated invariant, consistent with core indexing.
* Because `str` is immutable, the returned `str` may share the backing buffer internally; this is invisible and always safe.

> For zero-copy **byte-level** windows, use core slicing `s[lo..hi] -> slice<byte>` (`core/types.md` §6). `substring` is the codepoint-correct, owned-`str` counterpart.

---

## 4. Search

```bestie
fun str.contains(needle: str):    bool
fun str.startsWith(prefix: str):  bool
fun str.endsWith(suffix: str):    bool
fun str.indexOf(needle: str):     int ?   // codepoint index of first match, or absent
fun str.lastIndexOf(needle: str): int ?
fun str.count(needle: str):       int     // non-overlapping occurrences
```

`indexOf` / `lastIndexOf` return `int ?` (`option<int>`) — absence is explicit, never a `-1` sentinel.

```bestie
if (val i = path.indexOf("/")) {
    val head = path.substring(0, i)
}
```

---

## 5. Splitting and Joining

```bestie
fun str.split(sep: str):     list<str>
fun str.splitLines():        list<str>          // splits on \n / \r\n
fun str.splitWhitespace():   list<str>          // splits on runs of whitespace
fun list<str>.join(sep: str): str               // inverse of split
```

Rules:

* `split` on an empty separator is a **compile-time error** (ambiguous) — use `chars()` from core to iterate scalars instead.
* `split` never drops empty fields silently; `"a,,b".split(",")` yields `["a", "", "b"]`.
* `join` is an extension on `list<str>` so the pair reads naturally:

```bestie
val parts = "a,b,c".split(",")     // ["a", "b", "c"]
val back  = parts.join("-")         // "a-b-c"
```

---

## 6. Trimming and Padding

```bestie
fun str.trim():       str          // both ends
fun str.trimStart():  str
fun str.trimEnd():    str
fun str.padStart(width: int, pad: char): str
fun str.padEnd(width: int, pad: char):   str
```

Trimming removes leading/trailing Unicode whitespace. Padding adds `pad` until the string reaches `width` codepoints; a string already at or above `width` is returned unchanged.

---

## 7. Case Conversion

```bestie
fun str.toUpper(): str
fun str.toLower(): str
```

Rules:

* Uses **Unicode default case mapping** — no locale parameter.
* Locale-sensitive casing (e.g. Turkish dotless-i) is **not** provided here; it belongs in a dedicated internationalization library if/when one exists.

---

## 8. Other Transformations

```bestie
fun str.replace(old: str, new: str): str   // all non-overlapping occurrences
fun str.repeat(times: int): str            // times >= 0; 0 yields ""
fun str.reverse(): str                      // reverses by codepoint, not byte
fun str.isBlank(): bool                     // empty or only whitespace
fun str.charCount(): int                    // number of Unicode scalars (O(n))
```

`charCount()` is the codepoint count and is **O(n)** over UTF-8 — distinct from core's O(1) `byteSize()`. This is exactly why core exposes only `byteSize()` and defers a "length" with ambiguous units to this module.

---

## 8a. `char` Classification

Core `char` keeps only conversions — `toStr()` and `toInt32()` (`core/types.md` §2.6). Classification predicates are extensions declared here, for the same reason `str` parsing is: the set grows, and none of it is knowledge a programmer must have to know what Bestie is.

| Extension | Returns | Notes |
| --------- | ------- | ----- |
| `isAscii()` | `bool` | Scalar value in `0..=127` |
| `isDigit()` | `bool` | ASCII `0`–`9` |
| `isLetter()` | `bool` | Unicode letter category |
| `isAlphanumeric()` | `bool` | Letter or digit |
| `isWhitespace()` | `bool` | Unicode whitespace property |
| `isUpper()` / `isLower()` | `bool` | Unicode case category |
| `toUpper()` / `toLower()` | `char` | Unicode default case mapping (§7); no locale |

```bestie
import bestie.lib.strings

val c: char = 'A'
val ok = c.isAscii()
```

---

## 9. Relationship to Core and `StringBuilder`

| Concern | Where |
| ------- | ----- |
| Raw byte access, codepoint access, iteration, byte slicing | core `str` (`core/types.md` §3, §6) |
| Parsing, substring, search, split, trim, case, transforms | `bestie.lib.strings` (this document) |
| Efficient repeated concatenation | `StringBuilder` (`std-lib/util.md` §1) |

For building strings in a loop, prefer `StringBuilder` over repeated `+` or `join` to avoid intermediate allocations:

```bestie
import bestie.lib.util

var sb = StringBuilder.new()
for (part in parts) { sb.append(part) }
val result = sb.toStr()
```

---

## 10. Error Model

Parsing functions return the shared `ParseError`:

```bestie
errors ParseError {
    SyntaxError,
    MissingField,
    InvalidType,
    Overflow
}
```

Non-parsing operations do not fail: out-of-range substring indices **panic** (invariant violation), and search returns `option` rather than erroring.

---

## 11. Intentional Non-Features

* No implicit locale or culture
* No regular-expression engine (a separate library concern)
* No implicit trimming inside parse functions
* No mutation of `str` (it is immutable by definition)
* No `-1` sentinels — absence is always `option`

---

## 12. Summary

`bestie.lib.strings` is:

* **Extension-based** — keeps core `str` minimal while offering method-style ergonomics
* **Immutable** — every result is a new `str`
* **Codepoint-correct** — indices are Unicode scalars, never byte offsets that split characters
* **Explicit** — parsing is fallible and visible; absence is `option`; allocation is at the call site

It is the practical text-processing layer on top of Bestie's deliberately small core string type.

# bestie.lib.datetime — Date, Time, and Time Zones

This document defines **Bestie Standard Date & Time Library (`bestie.lib.datetime`)**.

Datetime belongs to **std-lib**, not `std-api`, because:

* Time is a logical concept, not an OS primitive
* The API is portable and deterministic
* OS clocks are accessed via `bestie.api.os`

---

## 1. Design Principles

1. **Explicit time zones**
2. **No implicit locale**
3. **Immutable values**
4. **Clear separation of concepts**
5. **No hidden global state**

---

## 2. Class Kinds and Ownership Rationale

Every type in this library has an explicit class kind. The rationale is:

| Type | Kind | Reasoning |
| ---- | ---- | --------- |
| `Instant` | `value class` | Two integer fields; always inlined; no identity |
| `Duration` | `value class` | Same shape and purpose as `Instant` |
| `Date` | `value class` | Three ints; lightweight; inlined everywhere |
| `Time` | `value class` | Four ints; same reasoning |
| `TimeZone` | `data class` | Contains `str` field — cannot be `value class`; structural equality is meaningful (same id + offset = same zone); immutable |
| `DateTime` | `data class` | Composite of value types; structural equality correct; immutable |
| `DateTimeFormatter` | `class` | Stateful (holds parse/format pattern); has identity |

**Field ownership:** All fields across `Instant`, `Duration`, `Date`, `Time`, `TimeZone`, and `DateTime` are value types — either primitives or other value/data classes. No field carries an `own` or `ref` qualifier. There is no heap allocation in any of these types; ownership accounting is not needed. `DateTimeFormatter` owns its internal pattern string by value.

---

## 3. Core Types

### 3.1 `Instant`

```bestie
value class Instant {
    seconds: int64
    nanos: int
}
```

* Absolute point in time (Unix epoch-relative)
* Time-zone independent
* Suitable for storage, comparison, and serialization
* Always copied — no identity semantics

Protocols implemented: `impl Equable<Instant>, Comparable<Instant>, Hashable<Instant>, Addable<Duration>, Subtractable<Duration>`

```bestie
val a: Instant = ...
val b: Instant = a + Duration.ofSeconds(60)   // Addable<Duration>
val d: Duration = b - a                        // see Arithmetic section
```

---

### 3.2 `Duration`

```bestie
value class Duration {
    seconds: int64
    nanos: int
}
```

* Represents elapsed time — no calendar semantics
* Always copied
* Negative durations are valid

Protocols implemented: `impl Equable<Duration>, Comparable<Duration>, Hashable<Duration>, Addable<Duration>, Subtractable<Duration>, Negatable`

Factory constructors (free functions):

```bestie
fun Duration.ofSeconds(s: int64): Duration
fun Duration.ofMillis(ms: int64): Duration
fun Duration.ofNanos(ns: int64): Duration
```

---

### 3.3 `Date`

```bestie
value class Date {
    year: int
    month: int    // 1–12
    day: int      // 1–31
}
```

* Calendar date — no time, no zone
* Month and day are 1-based
* Out-of-range values are a panic (invariant violation)

Protocols implemented: `impl Equable<Date>, Comparable<Date>, Hashable<Date>`

---

### 3.4 `Time`

```bestie
value class Time {
    hour: int     // 0–23
    minute: int   // 0–59
    second: int   // 0–59
    nanos: int    // 0–999_999_999
}
```

* Time of day — no date, no zone
* Out-of-range values are a panic

Protocols implemented: `impl Equable<Time>, Comparable<Time>, Hashable<Time>`

---

### 3.5 `DateTime`

```bestie
data class DateTime {
    date: Date        // value class — embedded by value, no own/ref
    time: Time        // value class — embedded by value, no own/ref
    zone: TimeZone    // data class — embedded by value, no own/ref
}
```

Time zone is **mandatory**. There is no `DateTime` without an explicit zone.

Protocols implemented: `impl Equable<DateTime>, Comparable<DateTime>, Hashable<DateTime>`

Note: `date`, `time`, and `zone` carry no `own` or `ref` qualifier because all three are value types (value class or data class). They are embedded inline in the `DateTime` layout with no heap allocation.

---

## 4. Time Zones

### 4.1 `TimeZone`

```bestie
data class TimeZone {
    val id: str          // IANA identifier, e.g. "America/New_York" or "UTC"
    val offset: Duration // base UTC offset — embedded by value, no own/ref
}
```

Rules:

* No implicit local zone
* UTC must be explicit: `TimeZone.utc()`
* Platform-specific zones (with DST rules) are loaded via `bestie.api.os` and wrapped in a `TimeZone`
* Structural equality: two `TimeZone` values with the same `id` and `offset` are equal
* `id` is a `str` (value type — no `own`/`ref` qualifier needed)
* `offset` is a `Duration` (value class — embedded inline)

Factory constructors (free functions):

```bestie
fun TimeZone.utc(): TimeZone
fun TimeZone.ofOffset(offset: Duration): TimeZone    // fixed offset zone, e.g. UTC+5:30
```

---

## 5. Conversions

Conversions are **free functions**, not methods. They are pure computations on value types with no associated state.

```bestie
fun toInstant(dt: DateTime): Instant
fun fromInstant(i: Instant, zone: TimeZone): DateTime
```

Rules:

* Conversions are explicit and loss-aware
* No implicit time zone injection
* `fromInstant` always requires an explicit zone

---

## 6. Arithmetic

Arithmetic is provided as **free functions** and via **protocol operator overloads** on the value types.

```bestie
// Free functions
fun add(i: Instant, d: Duration): Instant
fun diff(a: Instant, b: Instant): Duration
fun addDate(d: Date, days: int): Date
fun addTime(t: Time, d: Duration): Time

// Equivalent operator forms (via Addable/Subtractable impls)
val later: Instant = instant + duration     // Instant.add(Duration)
val elapsed: Duration = t2 - t1            // diff(t2, t1)
```

Rules:

* Calendar math is explicit — no hidden DST adjustments
* `diff` always returns a `Duration` (may be negative)
* Overflow in `Date`/`Time` arithmetic is a panic

**Class vs function:** All arithmetic operations are free functions or protocol impls on value types. No arithmetic belongs to a class because no state is required. `DateTimeFormatter` is the only class in this library; it exists because formatting requires a stateful pattern.

---

## 7. Formatting & Parsing

```bestie
errors DateTimeParseError {
    InvalidFormat,
    OutOfRange,
    InvalidZone
}

class DateTimeFormatter {
    fun format(dt: DateTime): str
    fun parseDateTime(text: str, zone: TimeZone): DateTime ! DateTimeParseError
}
```

`DateTimeFormatter` is a `class` (not `value class` or `data class`) because:

* It holds a stateful format pattern string
* It has identity (two formatters with the same pattern are distinct objects)
* Parsing is a stateful, fallible operation

Construction uses static factory methods (`@noNew`):

```bestie
val fmt = DateTimeFormatter.iso8601()
val fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")
```

```bestie
val text = fmt.format(dt)
val parsed = fmt.parseDateTime(text, zone) catch |err| { ... }
```

---

## 8. Immutability Guarantee

All datetime value types (`Instant`, `Duration`, `Date`, `Time`, `DateTime`, `TimeZone`) are:

* Immutable — no `var` fields
* Value-based — copied on assignment
* Thread-safe — safe to share across threads without synchronization
* Heap-free — no allocation involved in holding or passing them

`DateTimeFormatter` is a class with identity. It is immutable after construction (pattern is fixed) and may be freely shared.

---

## 9. Intentional Non-Features

This library intentionally avoids:

* Implicit system time
* Global clocks
* Locale-dependent formatting
* Overloaded constructors
* Magical "now" calls

System time comes from `bestie.api.os`.

---

## 10. Summary

`bestie.lib.datetime` is:

* Explicit
* Predictable
* Time-zone safe
* Free from OS leakage

It treats time as **data**, not magic.

Value types (`Instant`, `Duration`, `Date`, `Time`, `DateTime`, `TimeZone`) are all stack-allocated, inline-embedded, and carry no ownership burden. `DateTimeFormatter` is the single class — the only type that requires identity and internal state.

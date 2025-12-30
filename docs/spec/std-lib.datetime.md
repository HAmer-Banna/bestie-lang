# std-lib.datetime — Date, Time, and Time Zones

This document defines **Bestie Standard Date & Time Library (`std-lib.datetime`)**.

Datetime belongs to **std-lib**, not `std-api`, because:

* Time is a logical concept, not an OS primitive
* The API is portable and deterministic
* OS clocks are accessed via `std-api.os`

---

## 1. Design Principles

1. **Explicit time zones**
2. **No implicit locale**
3. **Immutable values**
4. **Clear separation of concepts**
5. **No hidden global state**

---

## 2. Core Types

### 2.1 `Instant`

```bestie
data class Instant {
    seconds: long
    nanos: int
}
```

* Absolute point in time
* Time-zone independent
* Suitable for storage and comparison

---

### 2.2 `Duration`

```bestie
data class Duration {
    seconds: long
    nanos: int
}
```

* Represents elapsed time
* No calendar semantics

---

### 2.3 `Date`

```bestie
struct Date {
    year: int
    month: int
    day: int
}
```

---

### 2.4 `Time`

```bestie
struct Time {
    hour: int
    minute: int
    second: int
    nanos: int
}
```

---

### 2.5 `DateTime`

```bestie
struct DateTime {
    date: Date
    time: Time
    zone: TimeZone
}
```

Time zone is **mandatory**.

---

## 3. Time Zones

### 3.1 `TimeZone`

```bestie
class TimeZone {
    id: str
    offset: Duration
}
```

Rules:

* No implicit local zone
* UTC must be explicit
* Platform zones are loaded via `std-api.os`

---

## 4. Conversions

```bestie
fun toInstant(dt: DateTime): Instant
fun fromInstant(i: Instant, zone: TimeZone): DateTime
```

Conversions are explicit and loss-aware.

---

## 5. Arithmetic

```bestie
fun add(i: Instant, d: Duration): Instant
fun diff(a: Instant, b: Instant): Duration
```

Rules:

* Calendar math is explicit
* No hidden DST adjustments

---

## 6. Formatting & Parsing

Formatting is delegated to `std-lib.format`.

```bestie
format.datetime.format(dt)
format.datetime.parse(str)
```

---

## 7. Immutability Guarantee

All datetime types are:

* Immutable
* Value-based
* Thread-safe

---

## 8. Intentional Non-Features

This library intentionally avoids:

* Implicit system time
* Global clocks
* Locale-dependent formatting
* Overloaded constructors
* Magical “now” calls

System time comes from `std-api.os`.

---

## 9. Summary

`std-lib.datetime` is:

* Explicit
* Predictable
* Time-zone safe
* Free from OS leakage

It treats time as **data**, not magic.

---

This document is **finalized**.

# Bestie Language — Memory Layout

This document defines how every Bestie value is **laid out in memory**: field order, padding, object headers, type words, tags, niches, and what the allocator adds around an object.

Layout is a first-class concern of the language, not a back-end detail. The compiler is **obligated** to produce the smallest valid representation for every type, and then the fastest one among equals. The programmer does not opt in, and there is nothing to opt out of.

Ownership, lifetime, and pointer rules live in `memory.md`. This document answers only **where the bytes go**.

---

## 1. Design Goals

1. **Zero per-object overhead by default.** An object carries its fields and nothing else unless it is part of a runtime-polymorphic hierarchy.
2. **Minimum size first, speed second.** Of the valid layouts, the compiler picks the smallest; among equally small layouts it picks the one with the cheapest access.
3. **No layout annotations.** There is no `@layout`, `@stable`, `@packed`, or `@align`. Good layout is the default, not a request. The one exception is FFI: `@repr(C)` in `bestie.api.foreign` matches a C header (§13).
4. **Deterministic.** The same type, compiler version, and target always get the same layout (§12).
5. **Closed world.** Every subtype of every `open class` is known when the program is linked. Bestie does not load subtypes at run time (§7.1).

---

## 2. What a Bestie Object Does Not Carry

In a managed runtime, every object has a header. Java's header holds GC mark and age bits, an identity hash, lock state, and a class pointer. Java's compact-header work reduces it to 8 bytes, with 4 as the target. Each part exists because the runtime needs per-object state. Bestie has no such state:

| Header content in managed runtimes | Why they need it | Bestie |
| ---------------------------------- | ---------------- | ------ |
| GC mark / age bits | Tracing collector | No garbage collection (`memory.md` §17) |
| Identity hash | Hash any object by identity | No identity hash. Hashing is a protocol (`Hashable`) over fields |
| Lock / monitor state | `synchronized` on any object | No per-object monitor. A `Lock` is a value you declare (`std-lib/concurrency.md` §6) |
| Class pointer | Run-time type of every object | Only for objects in a `virtual` hierarchy, and only 32 bits (§7) |
| Allocator chunk header | `free(p)` must find the size | None. Deallocation is **sized** (§10) |

The result, per class kind:

| Class kind | Hidden bytes per object |
| ---------- | ----------------------- |
| `class`, `data class`, `value class`, non-`virtual` `open class` | **0** |
| `open` / `abstract class` in a `virtual` hierarchy | **4** (type word, §7) |
| `sealed` `virtual` hierarchy | **1–2** (tag, §8); 4 for more than 65,536 types |
| tag-only `enum` | the tag is the value (§9.1) |
| `enum` with payloads, `T ?` | a tag, or **0** when a niche exists (§9, §11) |
| heap allocation of any of the above | **0** allocator header (§10) |

Hidden words, tags, and type words are never user-accessible fields. They cannot be named, read, or written through any typed access path (`memory.md` §10.1.3).

---

## 3. Size, Alignment, and Stride

Every type has three compile-time numbers:

| Query | Meaning |
| ----- | ------- |
| `sizeOf(T)` | Bytes a value of `T` occupies — up to the end of its last byte, **excluding** tail padding |
| `alignOf(T)` | Required address alignment — the largest alignment among its parts |
| `strideOf(T)` | Distance between consecutive elements of `T` in an array: `sizeOf(T)` rounded up to `alignOf(T)` |

For every primitive, `sizeOf(T) == strideOf(T)`. They differ only for aggregates whose last field leaves tail padding.

Separating size from stride is what lets an enclosing type use another type's tail padding (§5). The rules that follow from it:

* A value of `T` is copied or written as `sizeOf(T)` bytes, never `strideOf(T)`. Writing a whole `T` — by assignment, through `p.val = x`, or by move — never touches the bytes after `sizeOf(T)`, because they may belong to a neighbouring field.
* `array<T>`, `slice<T>`, and `p.offset(n)` step by `strideOf(T)`.
* A `T` that occupies a whole region (an array element, a heap allocation) owns its tail padding. A `T` embedded as a field does not.

All three queries are compile-time constants, valid in `const` initializers and `when` conditions (`lang.md` §15.5).

---

## 4. Field Placement

### 4.1 The algorithm

Source declaration order is **not** memory order. For every non-`@repr(C)` type, the compiler places fields like this:

1. **Pinned slots first.** The type word (§7) or sealed tag (§8) is placed at offset 0. For a subclass, every field the parent placed keeps the parent's offset (§6).
2. **Sort the remaining fields** by alignment, largest first. Break ties by size (largest first), then by declaration order.
3. **First fit.** Place each field at the lowest offset that satisfies its alignment and does not overlap any byte already placed. Holes left by pinned slots, by the parent, or by an earlier field's tail padding are filled before the object grows.
4. `sizeOf` is the end of the last placed byte. `alignOf` is the largest field alignment, or 1 for an empty type. `strideOf` follows from those two (§3).

The algorithm is deterministic. Declaration order is only a tie-breaker, so fields declared next to each other stay close in memory when nothing else decides.

### 4.2 Example

```bestie
class Packet {
    flag: bool
    id:   int64
    kind: uint8
    len:  int32
}
```

| Layout | Offsets | `sizeOf` |
| ------ | ------- | -------- |
| C, declaration order | `flag@0`, pad 7, `id@8`, `kind@16`, pad 3, `len@20` | 24 |
| Bestie | `id@0`, `len@8`, `flag@12`, `kind@13` | 14 (stride 16) |

### 4.3 Storage width of a field

* **`bool` is one byte** and is never bit-packed. Bit-packing would turn a write to one flag into a read-modify-write of its neighbours, which breaks `concurrency.md` §8.4 (distinct fields are distinct memory locations). Adjacent `bool` fields end up next to each other through step 2's ordering, not by sharing a byte.
* **Range-constrained types store the narrowest width that holds the range.** A field `as int in 0..=200` is stored in one byte (`compiler/compiler-architecture.md` §12). Its declared type is still the type the program sees; only storage narrows.
* **Unused bit patterns** of a narrowed field remain available as niches (§11).

---

## 5. Embedded Value Types

A `value class`, `data class`, `enum`, `tuple`, or fixed-size `array<T>` field is laid out **inline**: its bytes sit directly in the enclosing object, at the offset step 3 chose, with no pointer. A `value class` is always inline — on the stack, in an enclosing object, or in an array — and is never heap-allocated through `new()`.

An embedded value occupies `sizeOf` bytes, not `strideOf`, so the enclosing type can place fields in its tail:

```bestie
value class Sample {
    t:  int64
    ok: bool
}                      // sizeOf 9, alignOf 8, strideOf 16

class Reading {
    s:  Sample
    ch: uint8
}
```

| Layout | `Reading` |
| ------ | --------- |
| C (`sizeof(Sample)` is 16) | `s@0`, `ch@16` → 24 bytes |
| Bestie | `s@0..9`, `ch@9` → 10 bytes |

In `array<Sample>`, each element still takes `strideOf(Sample)` = 16 bytes. An array's elements must be uniformly aligned, so tail padding cannot be shared there.

---

## 6. Inheritance

A subclass's layout **extends** its parent's layout. A `ptr<Base>` to a `Derived` object must find every `Base` field at `Base`'s offsets.

1. Every field of the parent keeps the parent's offset in every subclass.
2. A subclass's own fields are placed by §4.1 around the parent's fields. They fill **the parent's interior holes and its tail** before growing the object.

Rule 2 is safe because a `class` is never copied by value (`memory.md` §13). C++ can only reuse padding in limited cases because assigning one base object to another would overwrite the derived fields in that padding. In Bestie, the only whole-object operations on a class are move and free, and both act on the most-derived object.

```bestie
open class Base {
    virtual fun run(): void
    a: int64
    b: int8
}

class Derived ext Base {
    override fun run(): void { ... }
    c: int16
    d: int8
    e: int32
}
```

| | Bestie | C++ (8-byte vptr) |
| - | ------ | ----------------- |
| `Base` | type word `@0..4`, `b@4`, `a@8` → 16 | vptr `@0`, `a@8`, `b@16` → 24 |
| `Derived` | adds `d@5`, `c@6`, `e@16` → 20 (stride 24) | adds fields after `Base` → 32 |

Copying `sizeOf(Base)` bytes of raw memory over a `Derived` object, through `ptr<byte>`, would overwrite `c` and `d`. That is raw memory work and is the programmer's responsibility, like every other raw copy (`memory.md` §15).

---

## 7. `virtual` Hierarchies — The 32-Bit Type Word

### 7.1 Closed world

Bestie does **not** load subtypes at run time. Every `open` or `abstract class` and every subtype that extends it is part of the program when it is linked. A dynamically loaded module cannot extend an `open class` from the host program; a plugin boundary uses FFI and function pointers (`std-api/foreign.md`), not inheritance.

This rule is what lets the type word be 32 bits, `is` run in constant time, and calls be devirtualized (§7.3).

### 7.2 Layout

An object of a hierarchy with at least one `virtual` method carries a **type word** at offset 0:

```
[ type word: uint32 | packed fields, filling bytes 4.. first ]
```

The type word is exactly 4 bytes on every target. It identifies the object's concrete class and is opaque: its value has no meaning to programs, and its encoding is the compiler's (`compiler/compiler-architecture.md` §12, *Type word and descriptors*).

An `open class` with no `virtual` methods has **no** type word. Where no runtime dispatch exists, the concrete type of every value is known statically (`memory.md` §10.1.4), so nothing needs to be stored in the object.

### 7.3 What the type word guarantees

From the type word alone, in constant time and without any per-object data beyond those 4 bytes:

* **Dispatch.** A `virtual` call reaches the most-derived override.
* **`is` at any depth.** `x is T` costs the same whether `T` is the direct class or a distant ancestor.
* **Freeing through a base.** `free()` / `freeDeep()` on a base-typed `own` value runs the most-derived `deinit` chain (`oop.md` §11.11) and releases the object's full size (§10).

Because the program is closed (§7.1), the compiler may also call a `virtual` method directly wherever it can prove only one implementation is reachable. The type word is still stored, because layout is fixed before the whole program is known (§15).

---

## 8. Sealed Hierarchies — The Compact Tag

A `sealed` `virtual` hierarchy (`oop.md` §12.1) replaces the type word with a **tag**: the smallest unsigned integer that distinguishes all its concrete types.

| Concrete types | Tag |
| -------------- | --- |
| 1 – 256 | `uint8` |
| 257 – 65,536 | `uint16` |
| more | `uint32` |

The tag is a **pinned field at offset 0**, not a separately padded prefix. Each concrete type's fields fill the bytes after it under §4.1:

```bestie
sealed Shape permits Circle, Rect

open class Shape { virtual fun area(): float64 }
class Circle ext Shape { r: float64 }
class Rect ext Shape { w: int16; h: int16; x: float64 }
```

| Type | Layout | `sizeOf` |
| ---- | ------ | -------- |
| `Circle` | tag `@0`, `r@8` | 16 |
| `Rect` | tag `@0`, `w@2`, `h@4`, `x@8` | 16 (a padded tag prefix would need 24) |

Rules:

* **Pre-order numbering.** If the base is concrete, it is tag 0. The permitted types follow in pre-order of the permit tree, so a nested sealed subtree occupies a contiguous range and `is` is a range check. For a single-level permit list this is simply declaration order.
* **Dispatch** goes straight from the tag to the target method, with no descriptor indirection.
* **Freeing through the base** finds the size and the `deinit` chain from the tag; nothing else is stored.
* Unused tag values are niches (§11).

---

## 9. Enums

### 9.1 Tag-only enums

A tag-only `enum` is its tag: an unsigned integer of the smallest fitting width. A two-variant enum is a byte with 254 spare patterns. With explicit discriminants (`enum E as T`), the tag has type `T` and the declared values (`oop.md` §3.3).

### 9.2 Enums with payloads

The tag is pinned at offset 0. **Each variant's payload is placed independently** by §4.1 around the tag, with no padded union region:

```bestie
enum Msg {
    Ping,
    Data(uint8, uint64),
    Ack(uint32)
}
```

| Variant | Layout | Bytes |
| ------- | ------ | ----- |
| `Ping` | tag `@0` | 1 |
| `Data` | tag `@0`, `uint8@1`, `uint64@8` | 16 |
| `Ack` | tag `@0`, `uint32@4` | 8 |
| **`Msg`** | `sizeOf` = largest variant, `alignOf` = largest alignment | **16** (a padded union would need 24) |

Bytes outside the active variant's fields are unspecified and are never read.

### 9.3 Enums with no tag at all

If one variant's payload has enough niches (§11), the other variants are encoded in those niche values and no tag is stored. This applies when every other variant is tag-only, or when the other variants' fields fit in bytes that do not overlap the niche field:

```bestie
type Score as uint8 in 0..=100

enum Grade { Pass(Score), Fail, Absent }    // 1 byte: Fail = 101, Absent = 102
enum Slot  { Empty, Full(own Buffer) }      // one pointer: Empty = zero address
```

---

## 10. Heap Allocation — No Allocator Header

`Type.new(...)` allocates exactly `sizeOf(T)` bytes, rounded up to the allocator's size class, at `alignOf(T)`. The default allocator stores **no metadata before or around the object**: no chunk header, no size word, no type word.

That works because every deallocation is **sized**: the compiler always knows the size being freed — from the static type, from the type word (§7.3), from the sealed tag (§8), or from a collection's stored capacity. How each case is lowered, and the default allocator's size classes, are the compiler's (`compiler/compiler-architecture.md` §12, *Sized deallocation*).

Consequences:

* An object costs its size rounded up to a size class — no header on top of that.
* Every `bestie.lib.allocators` allocator follows the same sized interface (`std-lib/allocators.md`). The `Debug` allocator may add redzones around allocations, which is its purpose; it is not the default.
* Memory obtained from C (`c.malloc`) is returned with C's `free`, never with Bestie's deallocation (`memory.md` §16).

---

## 11. Niches and `T ?`

A **niche** is a bit pattern that the type system guarantees is never a valid value of a type. `T ?` and tagged enums store their tag in a niche instead of in an extra byte wherever one exists.

### 11.1 Where niches come from

| Source | Niche values |
| ------ | ------------ |
| `bool` | 2 – 255 |
| `char` | Surrogates and values above U+10FFFF |
| Tag-only `enum`, sealed tag | Every unused tag value |
| Range-constrained type | Every pattern of the storage width outside the range |
| `own T` / `ref T` field, heap-allocated class value | The zero address |

`ptr<T>` provides **no** niche, because a raw pointer may hold the zero address (`memory.md` §8.7).

### 11.2 Niches are found recursively

The compiler searches the **fields** of a type for niches, not only the type itself. An aggregate has a niche if any field reachable inline in it has one:

```bestie
value class Entry { key: uint32; live: bool }   // sizeOf 5

Entry ?          // 5 bytes — "absent" is live = 2
User ?           // one pointer, when User is a heap class — "absent" is the zero address
```

Each level of `T ? ?` or each extra tagged variant consumes one niche value. Which niche is used is the compiler's choice (`compiler/compiler-architecture.md` §12), fixed per §15. When no niche exists, the tag is a pinned byte that fills a hole under §4.1. It is not a padded prefix.

### 11.3 Padding is never a niche

Padding bytes are not preserved: a whole-value write through `ptr<T>` may leave anything in them. A presence flag stored in padding would be corrupted by an ordinary write, so the compiler never uses padding as a niche.

The niche choice is invisible to programs. `T ?` always behaves as a two-variant type.

---

## 12. Concurrency and Layout

Layout never weakens the memory model in `concurrency.md` §8:

* Every field is its own memory location. Packing never merges two fields into one access, and narrowing (§4.3) never shares bytes between fields.
* A naturally aligned access of at most one machine word is a single access. Packing never misaligns a field: step 3 of §4.1 places every field at its own alignment.

**False sharing.** Two atomics written by different threads on the same cache line slow each other down. The compiler cannot see which fields are contended. Padding every `atomic<T>` to its own line would multiply the cost by every instance of every type that holds one, which contradicts minimum size by default. Bestie therefore does not insert cache-line padding implicitly, and there is no annotation to request it.

When contention is known, isolation is written as a **type**: `Isolated<T>` in `bestie.lib.concurrency` is a value type whose size and alignment are the target's cache-line size. A field of type `Isolated<atomic<int>>` occupies its own line wherever the compiler places it. As with `atomic<T>` orderings, the cost is visible in the source rather than chosen by default.

---

## 13. `@repr(C)` — The Only Fixed Layout

A type marked `@repr(C)` (`std-api/foreign.md` §5.2) is laid out in declaration order with the target C ABI's padding and alignment. For such a type, `sizeOf == strideOf` (C has no separate stride), no niche is taken from its fields, and the type cannot be part of a `virtual` hierarchy.

`@repr(C)` exists to match a C header. It is not a way to choose a layout for Bestie code.

---

## 14. What Bestie Deliberately Does Not Do

| Rejected | Reason |
| -------- | ------ |
| `@layout`, `@stable`, `@packed`, `@align` | Layout is the compiler's obligation, not a request. FFI uses `@repr(C)` |
| 8-byte vtable pointer | 4 bytes identify every class in a closed-world program (§7) |
| Bit-packed `bool` | Read-modify-write cost and races between distinct fields (§4.3) |
| Compressed 32-bit `own` / `ref` pointers | Requires one heap base, which conflicts with explicit allocators and the no-arena-sublanguage rule (`memory.md` §17). A 32-bit handle into one allocator is a library type, not a language default |
| Padding used as a niche | Padding is not preserved by writes (§11.3) |
| Struct-of-arrays layout chosen by the compiler | Breaks `.address()` on elements and `strideOf` arithmetic. SoA is a collection type, not a layout mode |
| Automatic cache-line padding | Its cost multiplies per instance; isolation is written as `Isolated<T>` (§12) |
| Loading subtypes at run time | Forces a full-pointer type word and open-world dispatch (§7.1) |

---

## 15. Stability Guarantees

* For a given compiler version and target, the same type always gets the same layout: offsets, size, alignment, stride, niche assignment.
* Layout is **not** stable across compiler versions or targets. Bytes that leave the process — files, network, shared memory between different builds — go through serialization or a `@repr(C)` type.
* Type-word values are stable within one linked program only.
* Sealed and enum tag values are stable within a compilation unit. Persist an `enum as T` discriminant (`oop.md` §3.3), never a compiler-assigned tag.
* `virtual` slot order is stable within a linked program.

---

## 16. Summary

* **Zero header.** Plain objects are their fields. Polymorphic objects add 4 bytes, sealed ones 1–2 bytes.
* **Zero allocator header.** Every free is sized, so the allocator stores nothing per object.
* **Minimum padding.** Fields are placed largest-alignment-first with first-fit hole filling, around pinned tags, inside parent holes, and into embedded values' tails.
* **Size ≠ stride.** Embedded values give their tail padding to the enclosing type, and arrays step by stride.
* **Tags cost nothing where a niche exists.** Niches are found recursively; padding is never used as one.
* **No annotations.** The only fixed layout is `@repr(C)`, for C.

Layout is not something the programmer tunes. It is something the compiler owes the programmer.

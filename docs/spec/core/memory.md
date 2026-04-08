# Bestie Language — Memory Management & Ownership Model

This document defines how memory is **laid out**, **owned**, **passed**, and **freed** in Bestie.

Memory management in Bestie is:

* Manual
* Explicit
* Deterministic
* Compile-time validated
* Systems-grade

Bestie does **not** hide memory.
It makes memory **predictable, inspectable, and intentional**.

There is **one memory model**, unified across system and backend domains.

---

## 1. Design Goals

The memory model exists to satisfy the following constraints:

1. **Native performance**
2. **Deterministic layout and lifetime**
3. **No garbage collection**
4. **No hidden unsafe behavior**
5. **Minimal cognitive overhead**
6. **Uniform rules for all domains**

Bestie rejects designs where:

* Safety is optional
* Allocation is implicit
* Different subsystems use different memory rules

Unsafe operations are allowed, but only through explicit syntax (`ptr<T>`, FFI, manual free) that is visible at the call site.

---

## 2. Why Bestie Has an Ownership System

Bestie was designed for **systems programmers and backend engineers**.

One of the earliest design questions was deceptively simple:

```bestie
class Student {
    val address: Address
    val course: Course
}
```

What should happen when we call:

```bestie
student.free()
```

Should it:

* Free `Student` only?
* Free `address`?
* Free `course`?
* Free everything transitively?

Different languages answer this implicitly.
Bestie refuses to.

### 2.1 The Core Problem

In real systems:

* Some fields are **owned**
* Some fields are **shared**
* Some fields are **borrowed**
* Some fields outlive the object
* Some must not be freed

Making this implicit leads to:

* Double frees
* Leaks
* Hidden sharing
* Runtime ownership bugs

### 2.2 The Explicit Answer

Bestie introduces **ownership qualifiers** to make intent unambiguous:

```bestie
class Student {
    val own address: Address
    val ref course: Course
}
```

This expresses **lifetime responsibility** directly in the type system.

* `Student` **owns** `address`
* `Student` merely **refers to** `course`

Now the behavior of `student.free()` is obvious.

---

## 3. Memory Regions

Bestie recognizes three conceptual memory regions.
These are **semantic**, not exposed as language modes.

### 3.1 Stack

Used for:

* Value types
* Inlined classes
* Function-local variables
* Temporaries

Properties:

* Deterministic lifetime
* Compile-time known layout
* Zero allocation cost

All values are stack-allocated unless explicitly moved elsewhere.

---

### 3.2 Heap

Used only when:

* `new()` is explicitly invoked
* Ownership must outlive the current scope
* Dynamic size requires it

Heap allocation is **never implicit**.

Example:

```bestie
own user = User.new()
```

---

### 3.3 Static / Read-only Memory

Used for:

* Constants
* Immutable literals
* Compile-time known values

Example:

```bestie
math.PI
```

---

## 4. Value Semantics vs Reference Semantics

### 4.1 Value Types

Includes:

* All primitives
* Tuples
* `data class`
* `value class`
*  collections

Properties:

* Copied on assignment
* Safe to pass across threads
* No ownership tracking needed

---

### 4.2 Reference Types

Reference and indirection semantics exist **only** via explicit constructs:

* `ref T` for borrowed references
* `ptr<T>` for raw addresses
* `own` qualifiers on bindings/fields for explicit ownership responsibility

There is no implicit reference behavior.

---

## 5. Ownership Qualifiers — `own` and `ref`

`own` and `ref` are **ownership qualifiers**, not containers and not types.

They describe **who is responsible for freeing memory**.

They may appear before or after `val` / `var`:

```bestie
val own address: Address
own val address: Address

var ref course: Course
ref var course: Course
```

All forms are equivalent.

---

### 5.1 `own` — Ownership

`own` means:

* This value has **exactly one owner**
* That owner is responsible for freeing it
* Ownership transfer is explicit
* Aliasing is forbidden

Example:

```bestie
val own user = User.new()
```

Rules:

* `own` values cannot be copied
* `own` values cannot be implicitly shared
* `own` values cannot be captured by lambdas
* Ownership transfer must be explicit
* Moved-from bindings are invalid and cannot be used
* Ownership back-links must use `ref` or `ptr`, not `own`

In ownership-validated code, every successful `new()` creates exactly one **ownership obligation**.
The compiler tracks that obligation flow-sensitively until it is discharged exactly once by one of these actions:

* `free()`
* `freeDeep()`
* transfer to another owner via `move`
* return to the caller as an owned result

If an ownership obligation reaches the end of its valid lifetime without being discharged, the compiler reports a **leak error**.
If the same obligation is discharged more than once, the compiler reports a **double-free error**.

This guarantee applies to code that stays within `own/ref` semantics. Explicit unsafe escapes through `ptr`, FFI, or trusted low-level constructs remain programmer responsibility.

---

### 5.2 Ownership Transfer

Ownership may be transferred via `move`:

```bestie
own a = User.new()
own b = move a
```

After move:

* `a` becomes invalid
* Any use is a compile-time error

---

### 5.3 `ref` — Borrowed Reference

`ref` means:

* This value is **not owned**
* Lifetime is controlled elsewhere
* The reference is temporary and scoped

Example:

```bestie
fun printUser(user: ref User) {
    print(user.name)
}
```

Rules:

* `ref` cannot outlive its owner
* `ref` cannot be stored
* `ref` cannot escape scope
* `ref` never implies ownership
* `ref` cannot cross thread boundaries
* Returning `ref` is valid only when borrowed from a value that outlives the caller

---

## 6. Function Calls and Ownership

### 6.1 Parameter Passing

Default: pass by value

Explicit:

* `ref` → borrow
* `own` → move

Example:

```bestie
fun process(own data: Data)
```

---

### 6.2 Return Values

When a function creates or produces a value, **someone must own it**.

Default return is by value.

```bestie
fun makeId(): int {
    return 42
}
```

When the returned expression is heap-allocated (for example via `new()`), ownership is transferred to the caller.

```bestie
fun createUser(): User {
    return User.new()
}
```

Equivalent explicit form (optional for readability):

```bestie
fun createUser(): own User
```

Rules:

* Value returns follow value semantics
* Returned objects are **owned by the caller**
* Returning an owned value transfers its ownership obligation to the caller
* Returning `ref` requires the owner to outlive the caller
* Returned `own` values cannot be duplicated by assignment

This eliminates:

* Escaping borrows
* Hidden heap sharing
* Lifetime ambiguity

---

## 7. Deallocation and Ownership Discharge

### 7.1 `free()`

```bestie
student.free()
```

`free()`:

* Frees **only the object itself**
* Does **not** recurse into owned fields
* Is valid only when all direct `own` fields have already had their ownership obligations discharged
* Is a compile-time error if live direct `own` fields would be leaked

---

### 7.2 `freeDeep()`

```bestie
student.freeDeep()
```

`freeDeep()`:

* Frees the object
* Recursively frees all `own` fields
* Skips all `ref` fields

The behavior of `freeDeep()` is **entirely determined at compile time**.

For deterministic ownership cleanup:

* Use `freeDeep()` for ownership trees
* Or free owned fields manually before `free()`

### 7.3 Ownership Accounting Rule

For ownership-validated code, Bestie enforces a simple invariant:

> every `new()` must be matched by exactly one ownership discharge.

The compiler maintains this accounting through moves, returns, container ownership, and explicit frees.
This is how Bestie avoids C-style leaks while staying simpler than Rust's full lifetime system.

---

### 7.4 Compiler Role — Report Only, No Implicit Cleanup

The compiler **does not silently insert `free()` or `freeDeep()` calls**. It is a static analysis and error reporting engine, not an implicit memory manager.

| Compiler behavior | When triggered |
| ----------------- | -------------- |
| **Leak error** | An `own` value's obligation is not discharged before it goes out of scope |
| **Double-free error** | An ownership obligation is discharged more than once |
| **Use-after-move error** | A moved-from binding is accessed |

These are compile-time errors. The program does not compile.

There is no RAII-style auto-drop at scope exit. The programmer must explicitly call `free()`, `freeDeep()`, or `move`. This is a deliberate design choice: implicit cleanup is hidden behavior, and Bestie refuses to hide memory.

```bestie
fun bad() {
    val own u = User.new()
    // ❌ compile error: ownership of 'u' is not discharged before scope exits
}

fun good() {
    val own u = User.new()
    u.freeDeep()    // ✅ explicit discharge
}
```

**Exception — construction failure cleanup:**

The one case where the compiler inserts cleanup automatically is fallible `init()` failure (see oop.md section 11.6). When a fallible `init()` returns an error, the compiler emits field-drop logic in reverse initialization order before returning the error to the caller. This is not silent: it is a specified protocol that is deterministic and part of the two-phase construction contract. The programmer declares the failure path with `! ErrorSet`; the compiler handles the cleanup mechanics for the already-initialized fields. This is not RAII — it is a well-defined, bounded exception to the no-implicit-cleanup rule, scoped entirely to the construction failure path.

---

## 8. Raw Pointers (`ptr<T>`)

### 8.1 Definition

`ptr<T>` represents a **raw memory address**, not ownership.

Rules:

* No ownership semantics
* No lifetime guarantees
* No implicit dereferencing
* Explicitly unsafe by design

---

### 8.2 Addressability Model

Ownership and addressability are **orthogonal**.

Having an address does **not** imply ownership.
Having ownership does **not** require exposing an address.

---

### 8.3 `.address()` — The Only Way to Get a Pointer

```bestie
value.address()
```

`.address()` returns a pointer to the **language-level memory representation**.

> `.address()` never lies about what it points to.

Return type depends on mutability:

| Value kind                 | Result         |
| -------------------------- | -------------- |
| Mutable value              | `ptr<T>`       |
| Immutable or `const` value | `ptr<const T>` |
| Runtime resource           | `ptr<const T>` |

---

### 8.4 Dereferencing Rules

Bestie has **no implicit dereferencing**.

```bestie
val v = p.val
p.val = 42
```

Const correctness is enforced:

```bestie
ptr<const T>  // read-only
ptr<T>        // mutable
```

---

### 8.5 Pointer Arithmetic

Allowed through explicit pointer APIs.

The compiler enforces bounds when provable.
When not provable, responsibility is explicit on the pointer-using code.

* Compile-time-known safe offsets are accepted
* Statically provable out-of-bounds is a compile-time error

```bestie
p.offset(2).val
```

---

## 9. Runtime Resources

Runtime resources (files, sockets, etc.) are addressable:

```bestie
val f = file.open()
val p = f.address()   // ptr<const file>
```

Meaning:

* Pointer refers to the handle object
* Not the OS kernel resource
* Handle is read-only
* Mutation through pointer is forbidden

Equivalent to `FILE*` in C.

---

## 10. User-Defined Types and Layout

User-defined classes and structs:

* Have real layout
* Have stable memory identity
* Are fully addressable

```bestie
val own u = User.new()
val p = u.address()   // ptr<User>
```

Inlining rules ensure:

* No hidden pointers
* Predictable field layout
* Cache-friendly access

---

## 11. Collections and Memory

### 11.1 Collection Ownership

Collections own their elements **only if element type is `own T`**.

```bestie
list<own User> users
```

* `users.freeDeep()` frees all users
* `users.free()` frees only the container

---

### 11.2 Copying Collections

* Value collections → deep copy
* Reference collections → reference copy
* Ownership collections → move only

Illegal:

```bestie
val a: list<own User>
val b = a   // compile-time error
```

---

### 11.3 Immutable & Concurrent Variants

```bestie
val l = list<int>.array.immutable.build()
```

Rules:

* Immutable collections may be shared across threads
* No mutation APIs
* Compile-time enforced

---

## 12. Stack vs Heap Optimization

* Value types prefer stack allocation
* Escape analysis applies
* Heap allocation is explicit

The compiler may inline, elide, reorder, and optimize as long as observable behavior is preserved.

---

## 13. Lifetime Enforcement

This section defines the **compiler mechanism** that enforces `ref` safety. The rules stated throughout this document (ref cannot outlive owner, cannot escape scope, cannot be stored) are enforced through **scope-based liveness analysis** within function bodies.

The key simplification that makes this tractable: **`ref` cannot be stored in fields**. This eliminates the need for lifetime parameters in types — the analysis stays entirely within function bodies.

---

### 13.1 Rule: Lexical Scope Containment

A `ref` is valid only within the scope where its source is alive. The compiler performs scope-based liveness analysis — no lifetime annotations required for this case.

```bestie
// ❌ compile error — x doesn’t outlive r
fun bad() {
    var r: ref int
    {
        val x = 42
        r = ref x    // x dies before r’s scope ends
    }
    print(r)
}

// ✅ fine — x outlives the inner block where r lives
fun good() {
    val x = 42
    {
        val r = ref x
        print(r)
    }
}
```

---

### 13.2 Rule: No Move While Borrowed

An `own` value cannot be moved while a `ref` to it is alive. The compiler tracks active borrows within the function body as a flow-sensitive analysis.

```bestie
val own u = User.new()
val r = ref u
val own v = move u    // ❌ compile error: u is borrowed by r
```

The borrow expires at the end of its enclosing scope. Moving is allowed once the borrow is gone:

```bestie
val own u = User.new()
{
    val r = ref u
    use(r)
}                      // r expires here
val own v = move u     // ✅ borrow has expired
```

---

### 13.3 Rule: Returning `ref` — Derivation and `from`

Returning a `ref` from a function is only valid when the returned reference derives from a value that outlives the caller. The compiler enforces this through three cases:

**Case 1 — Method returning from `this` (implicit, no annotation):**

```bestie
class User {
    val name: str
    fun getName(): ref str {
        return ref this.name    // ✅ derives from this — always safe
    }
}
```

**Case 2 — Single `ref` parameter (lifetime elision, no annotation):**

When a function has exactly one `ref` input, the returned `ref` is assumed to derive from it:

```bestie
fun first(xs: ref list<T>): ref T {
    return ref xs[0]    // ✅ only one ref input — derivation is unambiguous
}
```

**Case 3 — Multiple `ref` parameters (`from` annotation required):**

When derivation is ambiguous, `from` makes it explicit:

```bestie
fun longer(a: ref str, b: ref str): ref str from a, b {
    return if (a.len > b.len) ref a else ref b
}
```

`from a, b` tells the compiler: the returned `ref` may originate from either `a` or `b`. The caller must ensure both outlive the result.

**Returning a ref to a local is always a compile error:**

```bestie
fun bad(): ref int {
    val x = 42
    return ref x    // ❌ compile error: x doesn’t outlive the caller
}
```

---

### 13.4 Rule: No `ref` Across Thread Boundaries

The compiler rejects any `ref` passed into a thread closure at the point of thread creation:

```bestie
val x = 42
val r = ref x
threadOs.of(() => print(r))    // ❌ compile error: ref cannot cross thread boundary
```

Immutable values and `own` transfers (via `move`) are the safe alternatives. See `concurrency.md`.

---

### 13.5 What the Compiler Checks vs. What It Does Not

| Rule | How enforced |
| ---- | ------------ |
| `ref` doesn’t outlive source | Scope-based liveness analysis |
| No move while borrowed | Flow-sensitive borrow tracking within function body |
| `ref` not stored in fields | Type system |
| `ref` doesn’t cross thread boundaries | Thread closure analysis |
| Returned `ref` derives from valid input | `from` annotation + single-input elision |
| Every `new()` is discharged exactly once in safe ownership code | Ownership accounting over `own` values |
| `ptr<T>` bounds | Only when statically provable |

**Not checked by the compiler:**

* Aliasing of two `ref T` to the same value within a single thread — programmer responsibility, same as C
* Validity of `ptr<T>` beyond static bounds — programmer responsibility

This is not as strict as Rust’s exclusive `&mut` model, but it is significantly safer than C: no dangling refs for `own` values, no use-after-move, and no cross-thread ref races. The tradeoff favors simplicity and C-level performance.

---

## 14. Concurrency Rules

* `own` values cannot be implicitly shared
* `ref` cannot cross thread boundaries (compile-time enforced — see 13.4)
* `ptr<T>` crossing threads is explicit and programmer-owned

These rules ensure:

* Data-race freedom for code that stays within `own/ref` rules
* Deterministic memory behavior in safe code paths

---

## 15. Explicit Unsafe Boundary

Bestie exposes low-level operations directly instead of hiding them.

This boundary includes:

* Raw pointers and pointer arithmetic
* Manual deallocation
* FFI boundaries

Contract:

* Unsafe power is explicit in source code
* Ownership is never inferred from pointers
* The compiler rejects statically provable misuse
* Non-provable misuse remains the programmer’s explicit responsibility

---

## 16. FFI Pointer Rules

When interacting with foreign systems:

### Borrowed pointer (do NOT free)

```bestie
val p = os.getBuffer()
```

### Owned pointer (caller must free)

```bestie
val p = c.malloc(128)
c.free(p)
```

FFI APIs must document pointer ownership explicitly.

---

## 17. What This Model Rejects

Explicitly rejected designs:

* Garbage collection
* Implicit reference counting
* Arena sublanguages
* Unsafe blocks
* Borrow inference complexity
* Runtime lifetime tracking
* Lifetime parameters in type fields

---

## 18. Object and Tag Layout

This section defines the concrete in-memory layout for every class kind. These rules are stable and must be honored by the IR and codegen stages.

---

### 18.1 Regular `class` and `data class` (no virtual dispatch)

No object header. No type tag. No vtable pointer.

Fields are laid out in **declaration order**. The compiler may reorder fields for alignment padding reduction, provided it does so consistently and the result is observable only through `ptr<T>` field-offset arithmetic (which is always a programmer responsibility).

```
[ field_0 | field_1 | ... | field_n ]
```

`value class` follows the same rule. It is always inlined at the point of use — stack, or inline within an enclosing object — and is never heap-allocated through `new()`.

---

### 18.2 `open class` with `@virtual` — Vtable Layout

An object in a live `@virtual` hierarchy carries a **vtable pointer as its first field**. This is an implicit, hidden word prepended before all user-declared fields.

```
[ vtable_ptr | field_0 | field_1 | ... ]
```

The vtable is a read-only, statically allocated table of function pointers. Each `@virtual` method on the class occupies one slot, in declaration order. Slots are inherited from parent classes in the order they appear in the parent's vtable, followed by the subclass's own `@virtual` methods.

Vtable pointer size equals the platform pointer size (4 bytes on 32-bit, 8 bytes on 64-bit).

The compiler emits one vtable per concrete class. Abstract classes do not emit a vtable (they cannot be instantiated).

---

### 18.3 Sealed `@virtual` Hierarchy — Compact Tag Dispatch

When an `open class` hierarchy is declared `sealed` with a `permits` list, the compiler replaces the vtable pointer with a **compact type tag**.

Tag size: the smallest unsigned integer that can distinguish all permitted types.

| Permitted types | Tag type |
| --------------- | -------- |
| 1–255 | `uint8` |
| 256–65535 | `uint16` |
| > 65535 | `uint32` (rare) |

```
[ tag: uint8 | padding | field_0 | field_1 | ... ]
```

Tag values are compiler-assigned. The base type's tag = 0 if it is concrete; otherwise tags start at 0 for the first permitted subtype. The assignment is deterministic: permitted types in declaration order receive consecutive tag values starting at 0.

Dispatch on a sealed hierarchy lowers to a `switch` on the tag value, with direct calls to the target method — no vtable indirection.

---

### 18.4 `enum` — Tag-Only Variant

A tag-only `enum` (no payload) lowers to an unsigned integer of the smallest fitting size, using the same sizing rule as the sealed tag above.

```
[ tag: uint8 ]   // for ≤ 255 variants
```

The tag is the entire representation. No padding, no additional fields.

---

### 18.5 `enum` with Payload Variants — Discriminated Union

An `enum` with one or more payload variants uses a **discriminated union** layout:

```
[ tag | padding | payload_union ]
```

* `tag` — same sizing rule as above
* `padding` — inserted by the compiler to align the payload to the maximum alignment of any variant's payload type
* `payload_union` — sized to the largest payload variant, with each variant's fields laid out from the start of the union region

The total size of the discriminated union is:
```
sizeof(tag) + sizeof(padding) + sizeof(largest_payload)
```
rounded up to the alignment of the largest payload type.

Tag-only variants occupy the tag slot only; their payload region is undefined and not accessed.

---

### 18.6 `option<T>` — Niche Optimization

`option<T>` uses niche optimization where the type system guarantees a specific bit pattern is not a valid `T` value.

| `T` | Optimization |
| --- | ------------ |
| Reference or `own` heap-allocated type | Zero-address niche: `Not_Present` = all-zero word; `Present(x)` = non-zero address |
| `ptr<T>` (raw pointer) | No niche — `ptr<T>` may legitimately hold the zero address; `option<ptr<T>>` uses a tag |
| Primitive with a reserved bit pattern (e.g., `bool`) | Compiler-specific niche if unambiguous |
| All other types | Explicit tag: `[ tag: uint8 | padding | value ]` |

The niche optimization is invisible to user code. `option<T>` always behaves as a two-variant type; the layout is a compiler implementation detail.

---

### 18.7 Layout Stability Guarantees

* Field order (after compiler reordering) is stable across recompilations of the same source
* Vtable slot indices are stable within a compilation unit
* Tag values for sealed hierarchies and enums are stable within a compilation unit
* Cross-module ABI stability requires `@stable` annotation (not yet defined — tracked for a future spec revision)

---

## 19. Summary  <!-- formerly §18 — renumbered after §18 Object and Tag Layout was inserted -->

Bestie’s memory and ownership model is:

* Explicit, not magical
* Ownership-driven, not GC-driven
* Pointer-friendly, not pointer-hiding
* Deterministic, not runtime-driven
* Safe by default with explicit unsafe boundaries

`own` answers **who frees**
`ref` answers **who borrows**
`ptr<T>` answers **where in memory**
`from` answers **where a returned ref originates**

Memory is not managed *for* the developer.
Memory is managed **with** the developer — explicitly, safely, and predictably.

Nothing more.
Nothing less.

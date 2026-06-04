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

Value types are **copied on assignment**. Every binding holds its own independent copy. No heap allocation is required simply to hold the value.

| Kind | Value type? | Notes |
| ---- | ----------- | ----- |
| Primitives (`int`, `float64`, `bool`, `byte`, …) | ✅ | Stack-allocated, zero overhead |
| `tuple` | ✅ | Laid out inline, copied structurally |
| `value class` | ✅ | Copy-by-value, inlined at point of use |
| `data class` | ✅ | Copy-by-value, deeply immutable |
| `enum` (tag-only) | ✅ | Integer tag, no heap |
| `enum` (with payload) | ✅ unless contains `own` fields — see §4.3 | Discriminated union, stack-allocated |

Properties:

* Copied on assignment — each binding is independent
* Thread-safe by default — no shared mutable state possible through value semantics
* No ownership tracking required for the value itself

---

### 4.2 Reference Types

The following class kinds are **heap-allocated reference types**. A value of these kinds is a reference to a heap object, not the object itself.

| Kind | Reference type? | Notes |
| ---- | --------------- | ----- |
| `class` | ✅ | Default: heap-allocated, identity semantics |
| `open class` | ✅ | Same, plus vtable pointer in layout |
| `abstract class` | ✅ | Cannot be instantiated directly |

Reference types require explicit ownership qualification (`own` or `ref`) on bindings and fields to express who is responsible for freeing the heap object.

Reference and indirection semantics exist **only** via explicit constructs:

* `ref T` — a borrowed, scoped, non-owning reference
* `ptr<T>` — a raw address, unsafe, no lifetime or ownership guarantees
* `own` qualifier — explicit declaration of ownership responsibility

There is no implicit reference behavior. No class kind is automatically reference-counted or garbage-collected.

**`ptr<T>` is itself a value type** — a single machine word holding an address. It is copied by assignment, carries no ownership obligation, and is therefore a valid pointee and a valid element anywhere a value is allowed: as a field, as a collection element (`array<ptr<T>>` — see §11.4), or as the target of another pointer (`ptr<ptr<T>>` — see §8.6). Copying a `ptr<T>` duplicates the address, not the pointee; this aliasing is allowed precisely because raw pointers live in the explicit unsafe domain.

---

### 4.3 `enum` with `own` Payload — Move-Only Semantics

An `enum` variant that holds an `own` field becomes **non-copyable** — copying the enum would duplicate an ownership obligation, which is forbidden.

```bestie
enum Result {
    Ok(own Response)    // owns a heap-allocated Response
    Err(int)
}
```

Rules:

* An `enum` value containing an `own` field must be **moved**, not copied
* The ownership obligation follows the move — the source binding becomes invalid
* An `enum` where no variant carries an `own` field retains normal value (copy) semantics
* The compiler infers move-only status from the presence of `own` in any variant's payload

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

| Binding / value kind | Result         |
| -------------------- | -------------- |
| `val T` binding      | `ptr<T>`       |
| `const T` binding    | `ptr<const T>` |
| Borrowed `ref T`     | `ptr<const T>` |
| Runtime resource     | `ptr<const T>` |

A `val T` binding designates **writable storage**, so its address is a mutable `ptr<T>`. A `const T` binding designates an **immutable target**, so its address is a `ptr<const T>` — a *pointer to const*: the target can be read through it but **never mutated**. Borrowed references and runtime handles are likewise read-only. Immutable class kinds (for example `data class`) always yield `ptr<const T>` regardless of the binding — see §10.1.2.

> **`val` here is the binding axis, not the field axis.** The pointer's const-ness derives from the **binding** form (`val`/`var`/`const`/`ref`), i.e. whether the *name* designates writable or read-only storage. This is distinct from field-level `val` immutability: a `val` *field* inside a class cannot be mutated after construction. Field-level `val` governs write permission **through the resulting pointer on a per-field basis** (see §10.1.3) — it does not change whether `.address()` returns `ptr<T>` or `ptr<const T>`. For the two axes of the `val` keyword, see `core/lang.md` §4.2.

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

### 8.6 Pointer of Pointers and Const Propagation

A `ptr<T>` can point to any `T`, including another pointer. `ptr<ptr<T>>` is **not a special construct** — it is `ptr<U>` with `U = ptr<T>`. Nesting is arbitrary (`ptr<ptr<ptr<T>>>`), because a `ptr<T>` is itself a value (one machine word, no ownership — §4.2) and is therefore a valid pointee.

**Const-ness is independent at each level.** Each `const` qualifier applies to exactly one level of indirection:

| Type | Inner pointer slot | Final `T` |
| ---- | ------------------ | --------- |
| `ptr<ptr<T>>` | writable | writable |
| `ptr<ptr<const T>>` | writable | read-only |
| `ptr<const ptr<T>>` | read-only | writable |
| `ptr<const ptr<const T>>` | read-only | read-only |

**Dereferencing is explicit at every level** — Bestie never chases multiple hops implicitly:

```bestie
val p:  ptr<int>      = x.address()
val pp: ptr<ptr<int>> = p.address()

val inner: ptr<int> = pp.val         // one hop  → the stored pointer
val value: int      = pp.val.val     // two hops → the int

pp.val.val = 42     // write through to the int   — requires the int level non-const
pp.val     = q      // rebind the inner pointer    — requires the outer level non-const
```

**`.address()` needs no new rule** — the §10.1.2 binding rules apply with `T = ptr<...>`:

```bestie
val p: ptr<int>   = ...
val pp = p.address()     // ptr<ptr<int>>          (val binding → writable slot)

const c: ptr<int> = ...
val cc = c.address()     // ptr<const ptr<int>>    (const binding → read-only slot)
```

**Pointer arithmetic** on a `ptr<ptr<T>>` strides by one machine word (`sizeof(ptr<T>)`), since the elements are pointers.

**`option` and the niche (§18.6):** only the **outermost** level is examined. A `ptr<…>` may legitimately hold the zero address, so `option<ptr<ptr<T>>>` has **no niche** and uses an explicit tag, exactly like `option<ptr<T>>`.

---

### 8.7 Pointer Equality, Comparison, and the Zero Address

**Equality.** `ptr<T>` supports `==` and `!=`, comparing **raw addresses**. Two pointers are equal iff they hold the same address; pointee contents and provenance are not considered.

```bestie
val same = (p == q)      // true iff p and q hold the same address
```

**Ordering.** `<`, `<=`, `>`, `>=` are available and compare addresses numerically. Ordering pointers that derive from different allocations is permitted (it is the unsafe domain) but its meaning is the programmer's responsibility — only ordering within a single allocation or array is well-defined.

**The zero address.** Bestie has no `null` (see `fp.md` §4). Safe Bestie code never produces a zero-address `ptr<T>`: every pointer from `.address()` targets live storage and is non-zero. A zero address can enter a program only across an explicit boundary — `foreign` code or `@trusted` operations. For testing at that boundary:

```bestie
fun isZero(): bool      // true if the pointer holds the zero address
```

To model "a pointer that may be absent" in **safe** code, use `option<ptr<T>>` — the FFI layer maps a C `NULL` to `option.Not_Present` (see `foreign.md` §7). Reach for `isZero()` only inside `foreign` / `@trusted` code that handles raw addresses directly.

---

### 8.8 Pointer Casting and Alignment

Reinterpreting a pointer as a different pointee type is an **unsafe reinterpretation**, written explicitly:

```bestie
val pb: ptr<byte>   = p.cast<byte>()     // view the raw bytes of the pointee
val pu: ptr<uint32> = pb.cast<uint32>()  // reinterpret those bytes as a uint32
```

Rules:

* `cast<U>()` changes only the pointee type — the address value is unchanged. It performs **no conversion** of the pointed-to bytes (unlike numeric `as` conversion in `types.md` §2.1).
* Casting to `ptr<byte>` is always permitted (every type is byte-addressable). `byte`-level access is the canonical way to inspect raw storage.
* Casting **to a stricter alignment** than the address satisfies is undefined on access; alignment is the programmer's responsibility. The required alignment of `T` is compile-time known, and `p.isAligned<U>(): bool` tests an address before a stricter cast.
* **Removing `const`** (`ptr<const T>` → `ptr<T>`) is a const violation and is permitted **only under `@trusted`**. Adding `const` is always allowed.
* A reinterpret cast between unrelated pointee types is not, by itself, a compile-time error — it is the explicit unsafe boundary — but statically provable misuse (e.g., access beyond a known size) is still rejected per §8.5.

---

### 8.9 Function Pointers

A high-level callable value (`fn(...) -> ...` / `(...) -> ...`) uses the thin/fat representation defined in `fp.md` §6.2. A **raw function pointer** is the low-level address of executable code, used mainly for FFI:

```bestie
ptr<fn(int) -> int>      // raw code address of a function taking int, returning int
```

Rules:

* `.address()` on a **named function or a non-capturing lambda** yields a raw `ptr<fn(P) -> R>` — a single code pointer (the thin representation).
* A **capturing lambda is fat** (code + context); it has no single code address, so taking a raw function pointer to it is a **compile-time error**. Pass the callable value itself, or refactor to a non-capturing form.
* Calling **through** a raw `ptr<fn(...)>` is unsafe: there is no context word, no capture, and no lifetime guarantee. It exists for C interop — C function pointers map to this form (see `foreign.md`).
* A raw function pointer's pointee is always `const` — code is not writable through it.

---

### 8.10 Interior Pointers (Pointer to a Field)

`.address()` may be taken on a field to obtain a pointer **into** an object's storage:

```bestie
class Server { var port: int; val name: str }

val own s = Server.new(port: 8080, name: "prod")
val pp: ptr<int> = s.port.address()      // interior pointer at the port field's offset
pp.val = 9090                             // ✅ port is var
```

Rules:

* The result points at the field's storage slot, at its compile-time-known offset within the object.
* Const-ness follows the same two-axis rule as any other address: the **binding** axis sets the pointer's base const-ness (§10.1.2) and the **field**'s own `val`/`var` governs write-through (§10.1.3). A pointer to a `val` field rejects writes even when the pointer is non-const; a pointer derived from a `const` or `ref` binding is `ptr<const _>`.
* An interior pointer is valid only while the enclosing object is alive and not moved. If the object is freed or moved, the interior pointer dangles — programmer responsibility, identical to the ephemeral-address rule for value types (§10.1.2, §10.1.5).
* Interior pointers into a `value class` or other stack/inline value are ephemeral and must not be returned, stored, or sent across threads (§10.1.5).
* Pointer arithmetic from an interior pointer can walk adjacent fields; the compiler accepts compile-time-provable in-bounds offsets and rejects provable out-of-bounds (§8.5). Everything else is the programmer's responsibility.

---

### 8.11 Pointer Operations — Complete Reference

Every operation available on a `ptr<T>` value, in one place. This is the authoritative surface; changes to pointer behavior are made here.

| Operation | Result | Section | Notes |
| --------- | ------ | ------- | ----- |
| `p.val` | `T` | §8.4 | Dereference (read). No implicit deref. On `ptr<const T>`, read-only. |
| `p.val = x` | — | §8.4 | Dereference (write). Forbidden on `ptr<const T>` and on `val` fields (§10.1.3). |
| `p.address()` | `ptr<ptr<T>>` | §8.6 | Address of the pointer's own storage slot. Const-ness per binding (§10.1.2). |
| `p.offset(n)` | `ptr<T>` | §8.5 | Pointer arithmetic; strides by `sizeof(T)`. Provable OOB is a compile error. |
| `p.cast<U>()` | `ptr<U>` | §8.8 | Reinterpret pointee type; address unchanged, no byte conversion. |
| `p.isAligned<U>()` | `bool` | §8.8 | True if the address satisfies `U`'s alignment. |
| `p.isZero()` | `bool` | §8.7 | True if the address is the zero address (FFI / `@trusted` boundary). |
| `a == b`, `a != b` | `bool` | §8.7 | Address equality. Provenance not considered. |
| `a < b`, `<=`, `>`, `>=` | `bool` | §8.7 | Address ordering; well-defined only within one allocation. |

Const and provenance rules:

* `cast<U>()` that **removes `const`** (`ptr<const T>` → `ptr<T>`) requires `@trusted`; adding `const` is always allowed.
* No operation in this table allocates, frees, or transfers ownership — `ptr<T>` is pure indirection (§4.2). Freeing is always an explicit, separate call by whatever owns the memory.
* `copy(p)` / `deepCopy(p)` both duplicate the **address only** — raw pointers are not followed (see `std-lib/util.md`).

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

## 10.1 Class Kinds — Ownership, Addressability, and Pointer Rules

This section defines the complete relationship between class kinds and the memory model. It answers: which kinds can carry `own` fields, what `.address()` returns, what `ptr<T>` mutation is permitted, and how protocols interact with memory.

---

### 10.1.1 `own` and `ref` Field Rules per Class Kind

#### Two distinct uses of `ref`

`ref` plays two completely different roles in Bestie. Conflating them produces the contradiction. They must be kept separate:

| Use | Syntax | What it means | Compiler enforcement |
| --- | ------ | -------------- | -------------------- |
| **Field ownership qualifier** | `val ref course: Course` (in a class body) | This field is non-owning. Do not free it. | Ownership accounting only — `free()`/`freeDeep()` skip this field |
| **Local borrow** | `val r = ref x` or `fun f(x: ref Course)` | A scope-bounded borrow of an existing value | Scope-based liveness analysis — cannot outlive its source |

These are not the same thing. The rule in §13 — "a locally-obtained `ref` borrow cannot be assigned to a class field" — applies exclusively to the local borrow form. It has nothing to say about field ownership qualifiers.

---

#### `ref` as a field ownership qualifier (class fields only)

Declaring a field with `ref` as its ownership qualifier is the mechanism for expressing "this class borrows this object — someone else owns it and is responsible for freeing it."

```bestie
class Student {
    val own address: Address    // Student owns address — freed with Student
    val ref course: Course      // Student borrows course — NOT freed with Student
}
```

This is the foundational use case established in §2. `student.freeDeep()` frees `address` and skips `course`. The compiler does not track the lifetime of `course` — that is the programmer's responsibility (the Course must outlive the Student).

**`ref` field qualifiers are permitted only on heap-allocated reference-type class kinds.** They are forbidden on value types because:

* `value class` — value classes are fully independent copies. Adding a non-owning reference creates an external lifetime dependency that breaks copy safety. If you copy the value class, both copies point to the same object, and neither owns it — the original owner can free the object while both copies still hold the pointer.
* `data class` — deeply immutable means the entire object graph is stable and self-contained. A `ref` to an external mutable object breaks this guarantee — the referenced object could be freed or changed through another owner.
* `enum` payload — same reasoning as `value class`; payload variants have value semantics.

For value-type fields with no `own` or `ref` qualifier, the compiler always applies **copy semantics** — the value is embedded inline or copied by value. No ownership tracking is needed.

---

#### Ownership qualifier rules per class kind (full table)

| Class kind | `own` fields | `ref` field qualifier | `ptr<T>` fields |
| ---------- | ------------ | --------------------- | --------------- |
| `value class` | ❌ Copy would duplicate ownership | ❌ Breaks copy-safety and lifetime independence | ✅ Allowed (unsafe) |
| `data class` | ❌ All fields are `val`; deep immutability | ❌ External ref breaks deep immutability | ✅ `ptr<const T>` only (unsafe) |
| `enum` (tag-only) | ❌ No payload | ❌ No payload | ❌ No payload |
| `enum` (payload) | ✅ Makes enum move-only (§4.3) | ❌ Value-type semantics | ✅ Allowed (unsafe) |
| `class` | ✅ | ✅ Non-owning field; not freed | ✅ Allowed (unsafe) |
| `open class` | ✅ | ✅ Non-owning field; not freed | ✅ Allowed (unsafe) |
| `abstract class` | ✅ | ✅ Non-owning field; not freed | ✅ Allowed (unsafe) |

`ptr<T>` fields are always in the programmer's responsibility domain — the compiler enforces no ownership semantics on raw pointers. The class that holds a `ptr<T>` field is responsible for freeing the pointed-to memory through explicit `release()`, `free()`, or similar calls.

`data class` allows only `ptr<const T>` fields — not `ptr<T>` — because a mutable raw pointer field would allow mutation through the pointer, violating deep immutability.

---

#### Ownership qualifier requirement for reference-type fields

For fields whose type is a reference type (`class`, `open class`, `abstract class`), the `own` or `ref` qualifier is **required**. The compiler cannot determine freeing semantics from the type alone.

```bestie
class Team {
    val own leader: Employee       // ✅ required — Team owns this Employee
    val ref department: Department // ✅ required — Team borrows this Department
    val member: Employee           // ❌ compile-time error — ambiguous: own or ref?
}
```

For fields whose type is a value type (`value class`, `data class`, `enum`, primitives), no qualifier is needed — the value is always embedded by copy.

---

#### What `ref` field qualifier does NOT mean

* It does NOT create a compile-time-tracked borrow — lifetime enforcement applies only to local `ref` borrows (§13)
* It does NOT prevent the programmer from freeing the referenced object while the field still points to it — that is programmer responsibility
* It does NOT produce a `ref T` type — the field's type is still `Course`, not `ref Course`. `ref` is a qualifier, not a type constructor.

---

### 10.1.2 `.address()` Return Type Rules

`.address()` returns a pointer to the **language-level memory representation** of the value. The const-ness of the returned pointer is determined by two rules applied in order:

**Rule 1 — Class kind immutability is absolute:**

| Class kind | `.address()` always returns |
| ---------- | --------------------------- |
| `data class` | `ptr<const T>` — regardless of binding mutability. Deep immutability is a type-level guarantee. |
| `value class` with all `val` fields | `ptr<const T>` |
| `value class` with any `var` field | Follows Rule 2 |
| `class`, `open class`, `abstract class` | Follows Rule 2 |
| `enum` | `ptr<const T>` — enums are value types; tag and payload are not mutated through pointers |

**Rule 2 — Binding mutability determines const-ness for mutable types:**

The const-ness of the returned pointer reflects whether the target may be mutated. A `val T` binding yields `ptr<T>` — the address of writable storage. A `const T` binding yields `ptr<const T>` — a pointer to const, through which the target cannot be mutated. `var T` and `own T` likewise yield `ptr<T>`; a borrowed `ref T` yields `ptr<const T>`.

> **Binding axis vs. field axis.** Rule 2 reads the **binding** form (`val`/`var`/`const`/`ref`) — the `val` here means "the binding cannot be rebound," not "the field cannot be mutated." It governs the *pointer's* const-ness only. Field-level `val` is a separate concept: it controls which fields may be written *through* the resulting pointer, per field, and is applied in §10.1.3. A non-const `ptr<T>` can still refuse a write to a `val` field.

| Binding | `.address()` returns |
| ------- | -------------------- |
| `val T` | `ptr<T>` — address of writable storage |
| `var T` | `ptr<T>` |
| `const T` | `ptr<const T>` — pointer to const; the target cannot be mutated through it |
| `own T` (heap-allocated class) | `ptr<T>` — owner has full access |
| `ref T` (borrowed reference) | `ptr<const T>` — a borrow cannot produce a mutable pointer; mutation through borrowed address would bypass the borrow rules |

```bestie
// class — val binding → mutable pointer
val own user = User.new()
val p1 = user.address()          // ptr<User>

// class — val binding → mutable pointer
val u2: User = someUser
val p2 = u2.address()            // ptr<User>

// class — const binding → const pointer
const u3: User = someUser
val p3 = u3.address()            // ptr<const User>  — const target cannot be mutated through it

// data class — always const regardless of binding
var dt: DateTime = DateTime.new(...)
val p4 = dt.address()            // ptr<const DateTime>  — var binding but data class is immutable

// value class — var binding → mutable pointer
var pt: Point = Point.new(1, 2)
val p5 = pt.address()            // ptr<Point>

// ref — always const pointer
fun inspect(r: ref User) {
    val p6 = r.address()         // ptr<const User>
}
```

**Ephemeral address warning for value types:**

Value types (`value class`, `data class`, primitives) live on the stack or inline within another object. Their `.address()` is valid only within the scope of the binding. If the binding goes out of scope or is moved, the pointer becomes dangling. This is not a compile-time error — it is programmer responsibility, consistent with all other `ptr<T>` usage. The compiler does not insert bounds checks or lifetime enforcement on `ptr<T>`.

---

### 10.1.3 Mutation Through `ptr<T>` — What Can Be Changed

Mutation through a pointer follows the **field's own declared mutability**, not just the pointer's const-ness.

**Rule: `ptr<T>` gives access governed by field mutability.**

```bestie
class Server {
    var port: int        // mutable field
    val name: str        // immutable field
}

val own s = Server.new(port: 8080, name: "prod")
val p: ptr<Server> = s.address()

p.val.port = 9090    // ✅ allowed — port is var
p.val.name = "dev"   // ❌ compile-time error — name is val
```

**Rule: `ptr<const T>` forbids all mutation through the pointer, regardless of field mutability.**

```bestie
val s2: Server = ...
val p2: ptr<const Server> = s2.address()

p2.val.port = 9090   // ❌ compile-time error — ptr<const T> forbids all writes
```

**`data class` through `ptr<T>`:** Because all `data class` fields are implicitly `val`, even a `ptr<DataClass>` (non-const) cannot mutate any field — all field assignments are rejected at compile time. The const-ness of the pointer is effectively irrelevant for data classes; both `ptr<T>` and `ptr<const T>` give read-only access.

```bestie
var dt: DateTime = DateTime.new(...)
val p: ptr<DateTime> = dt.address()    // ptr<DateTime>, not ptr<const DateTime>

p.val.date = Date.new(...)             // ❌ compile-time error — date is val in data class
```

**`open class` vtable pointer:** The hidden vtable pointer prepended to `open class` objects (see §18.2) is **never user-accessible**. It does not appear as a field name, cannot be read, and cannot be overwritten through any pointer. The compiler guarantees the vtable pointer is read-only from all access paths, including `ptr<OpenClass>`. Overwriting the vtable through `ptr<byte>` and raw offsets is possible (it is unsafe code) but is programmer-exclusive responsibility.

---

### 10.1.4 Protocols and Memory

Protocols are **zero-cost compile-time abstractions**. They have no memory footprint of their own.

**Static protocol dispatch (default):**

A variable declared as a protocol type stores the concrete object directly, with the concrete object's layout. The protocol type is a compile-time view — not a separate allocation, not a fat pointer.

```bestie
val p: Printable = Circle.new(radius: 5)
// p stores a Circle — layout is Circle's layout
// no boxing, no fat pointer, no vtable in p itself
```

The concrete type must be statically known at the point of assignment. This means:

* `val p: Printable = Circle.new()` — ✅ concrete type is known at compile time
* `list<Printable>` containing mixed `Circle` and `Rectangle` — ❌ requires runtime polymorphism; use a sealed `open class` hierarchy with `@virtual` instead

`.address()` on a protocol-typed variable resolves to the concrete type's pointer:

```bestie
val p: Printable = circle
val addr = p.address()   // ptr<const Circle> — concrete type is known at compile time
```

**Dynamic dispatch via `@virtual`:**

When dynamic dispatch is needed (elements of different concrete types), use `@virtual` methods and an `open class` hierarchy. The vtable pointer lives inside the object (see §18.2), not in a separate indirection layer. There is no fat-pointer protocol mechanism in Bestie.

**Protocols have no fields, no size, no allocation.** Attempting to store a protocol as a standalone value without a concrete type is a compile-time error.

---

### 10.1.5 `value class` — Addressability Caution

`value class` is designed for inline embedding and stack use. Calling `.address()` on a `value class` is valid but carries the following restrictions:

* The returned pointer is valid only within the scope of the binding — it is **not safe** to return, store in a field, or pass across thread boundaries
* If the binding is a function parameter passed by value, `.address()` gives the address of the local copy — not the caller's storage
* Assigning to the binding after taking its address does **not** invalidate the pointer (the address of the storage slot is stable within the scope), but the content at that address changes immediately

```bestie
fun bad(p: Point): ptr<Point> {
    return p.address()    // ❌ compile-time error — returning ptr to local value type
}

fun ok(p: Point) {
    val addr = p.address()    // ✅ valid within this scope only
    use(addr)
}
```

`own` fields are forbidden on `value class`, so there is never an ownership complication from taking an address of a `value class` field.

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

A collection **always owns its backing buffer**, regardless of element type. Two consequences follow from the core ownership rules (§5.1 — ownership transfer must be explicit) and the no-hidden-allocation pillar:

* Bare assignment `val b = a` of an existing collection is **never** a silent copy (that would be a hidden O(n) allocation) **nor** a silent move (moves must be explicit).
* Duplicating or transferring a collection is therefore always **explicit** — `move`, `copy()`, or `deepCopy()`.

| Operation on a collection `a` | Result |
| ----------------------------- | ------ |
| `val b = a` | ❌ compile-time error — choose `move`, `copy`, or `deepCopy` |
| `val b = move a` | ownership transferred; `a` becomes invalid; no allocation |
| `copy(a)` | new container; behavior per element kind (below) |
| `deepCopy(a)` | new container; owned elements recursively duplicated |

What `copy()` / `deepCopy()` do, per element kind:

| Element kind | `copy(a)` | `deepCopy(a)` |
| ------------ | --------- | ------------- |
| value elements (`list<int>`) | new buffer, elements copied | identical to `copy` |
| reference / borrow (`list<ref T>`) | new buffer, **same borrows** (aliased) | same |
| ownership (`list<own T>`) | ❌ forbidden (would duplicate ownership) | new buffer, **each element deep-copied** |
| raw pointers (`list<ptr<T>>`) | new buffer, **same addresses** (aliased) | same (raw pointers not followed) |

```bestie
val a: list<int> = {1, 2, 3}

val b = a            // ❌ compile-time error — collection owns its buffer
val b = move a       // ✅ transfer; a is now invalid
val c = copy(a)      // ✅ explicit independent duplicate (allocation is visible)

val o: list<own User> = ...
val q = deepCopy(o)  // ✅ new container; each User deep-copied
val r = copy(o)      // ❌ forbidden — would duplicate ownership of elements
```

Duplication semantics are defined in full in `std-lib/util.md` §8. Note that for `list<ptr<T>>`, `copy()` produces a new buffer holding the **same addresses** — the container is duplicated, the pointees are aliased, because `ptr<T>` carries no ownership (§4.2).

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

### 11.4 Containers of Pointers

Any collection may hold raw pointers as elements — `array<ptr<T>>`, `list<ptr<T>>`, `map<K, ptr<V>>`, and so on — because `ptr<T>` is a value-type element (one word — §4.2). Nesting (`array<ptr<array<int>>>`) and const element forms (`array<ptr<const T>>`) are equally valid.

**A pointer-element container owns nothing it points through.** `ptr<T>` carries no ownership, so the container is responsible only for its own backing buffer. This is the decisive difference from owning and borrowing element kinds:

| Element type | What the container owns | `freeDeep()` | Copy semantics |
| ------------ | ----------------------- | ------------ | -------------- |
| `list<own T>` | the elements | frees buffer **and every element** | move-only (no copy) |
| `list<ref T>` | nothing (borrows) | frees buffer only | borrow |
| `list<ptr<T>>` | nothing (raw addresses) | frees buffer only — **pointees are never freed** | shallow copy (aliasing) allowed |

Consequences, all in the explicit unsafe domain:

* **Freeing pointees is manual.** `freeDeep()` on a `list<ptr<T>>` behaves like `free()` with respect to the pointees — it reclaims the backing buffer, not the pointed-to memory. To free the targets, iterate and release each explicitly **before** freeing the container.
* **Copying aliases.** `val b = a` on a `list<ptr<T>>` is a shallow copy: `a` and `b` hold the **same addresses**. This is permitted (unlike `list<own T>`, which is move-only — §11.2) and is intentional — it matches C-style pointer arrays. The programmer must not free through one alias while the other is still in use.

```bestie
val arr: array<ptr<Node>> = ...

val n0: ptr<Node> = arr[0]      // a copy of the stored address
arr[0].val.value = 9            // write through to the pointee (ptr non-const)
arr[0] = other                  // replace the stored pointer (container mutable)
```

**Concurrency.** A container of `ptr<T>` shared across threads is programmer-owned: the compiler provides no data-race protection on raw pointers (§14), unlike `own` / `ref` element containers. The `concurrent` variant guards only the container's own structure; the pointees remain unguarded.

**Value- / data-class elements that hold pointers** follow the field rules of §10.1.1: a `list<C>` whose `data class C` has a `ptr<const T>` field is fine; a mutable `ptr<T>` field inside a `data class` is rejected.

---

## 12. Stack vs Heap Optimization

* Value types prefer stack allocation
* Escape analysis applies
* Heap allocation is explicit

The compiler may inline, elide, reorder, and optimize as long as observable behavior is preserved.

---

## 13. Lifetime Enforcement

This section defines the **compiler mechanism** that enforces `ref` safety. The rules apply exclusively to **local `ref` borrows** — values obtained via `ref` expressions or `ref` parameters (the local borrow form of `ref`). Field ownership qualifiers (`val ref course: Course` in a class declaration) are a separate concept and are not subject to these rules — see §10.1.1.

The rules in this section are enforced through **scope-based liveness analysis** within function bodies.

The key simplification that makes this tractable: **a locally-obtained `ref` borrow cannot be assigned to a class field**. The `ref` qualifier on a class field *declaration* is not an assignment of a local borrow — it is a compile-time ownership annotation. The restriction is: you cannot take a `ref` borrow of a local or parameter value and store that borrow into a field whose lifetime exceeds the borrow's scope. This eliminates the need for lifetime parameters in types — the analysis stays entirely within function bodies.

```bestie
// ❌ What this rule forbids — storing a local borrow into a long-lived field:
fun bad(c: ref Course): Student {
    return Student.new(course: c)
    // ERROR: c is a locally-scoped ref borrow; it cannot populate a field
    //        that outlives this function call
}

// ✅ What is allowed — ref as a field ownership qualifier at declaration time:
class Student {
    val own address: Address
    val ref course: Course      // OK: ownership qualifier, not a stored local borrow
}

// ✅ Constructing with a reference-type value (no local ref involved):
val own c = Course.new(...)
val own s = Student.new(address: addr, course: c)
// Student holds a non-owning reference to c; c is still owned by this scope
```

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

`ptr<T>` composes freely — it **nests** (`ptr<ptr<T>>`, with per-level const), lives **in containers** (`array<ptr<T>>`, which own nothing they point through), and **casts** (`cast<U>()`, `@trusted` to drop const) — always as raw, unowned, explicit indirection. Pointer-of-pointers, pointer elements, pointer casts, function pointers, and interior pointers are all the same single primitive used compositionally, never new machinery.

Memory is not managed *for* the developer.
Memory is managed **with** the developer — explicitly, safely, and predictably.

Nothing more.
Nothing less.

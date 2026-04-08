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

## 10.1 Class Kinds — Ownership, Addressability, and Pointer Rules

This section defines the complete relationship between class kinds and the memory model. It answers: which kinds can carry `own` fields, what `.address()` returns, what `ptr<T>` mutation is permitted, and how protocols interact with memory.

---

### 10.1.1 `own` and `ref` Field Rules per Class Kind

`ref` fields are **forbidden in all class kinds** without exception. A `ref` cannot escape its source's scope; storing it in a field would require lifetime parameters on types, which Bestie explicitly rejects (see §13).

`own` fields are permitted only on class kinds that are heap-allocated reference types and support identity semantics. They are forbidden on value types because copying a value type would duplicate the ownership obligation.

| Class kind | `own` fields | `ptr<T>` fields | Reason |
| ---------- | ------------ | --------------- | ------ |
| `value class` | ❌ Forbidden | ✅ Allowed (unsafe) | Copy-by-value would produce two owners of the same heap object |
| `data class` | ❌ Forbidden | ✅ `ptr<const T>` only (unsafe) | Deeply immutable — no ownership to transfer; copying would duplicate obligation |
| `enum` (tag-only) | ❌ N/A | ❌ N/A | No payload fields |
| `enum` (payload) | ✅ Allowed | ✅ Allowed (unsafe) | Makes enum move-only — see §4.3 |
| `class` | ✅ Allowed | ✅ Allowed (unsafe) | Standard reference type; owns sub-objects |
| `open class` | ✅ Allowed | ✅ Allowed (unsafe) | Same as `class` |
| `abstract class` | ✅ Allowed | ✅ Allowed (unsafe) | Fields visible to concrete subclasses |

`ptr<T>` fields are always in the programmer's responsibility domain — the compiler enforces no ownership semantics on raw pointers. The class that holds a `ptr<T>` field is responsible for freeing the pointed-to memory through explicit `release()`, `free()`, or similar calls.

`data class` specifically allows only `ptr<const T>` fields — not `ptr<T>` — because the type is deeply immutable and a mutable raw pointer field would allow mutation through the pointer, violating the immutability contract.

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

| Binding | `.address()` returns |
| ------- | -------------------- |
| `val T` | `ptr<const T>` |
| `var T` | `ptr<T>` |
| `own T` (heap-allocated class) | `ptr<T>` — owner has full access |
| `ref T` (borrowed reference) | `ptr<const T>` — a borrow cannot produce a mutable pointer; mutation through borrowed address would bypass the borrow rules |

```bestie
// class — mutable binding → mutable pointer
val own user = User.new()
val p1 = user.address()          // ptr<User>

// class — immutable binding → const pointer
val u2: User = someUser
val p2 = u2.address()            // ptr<const User>

// data class — always const regardless of binding
var dt: DateTime = DateTime.new(...)
val p3 = dt.address()            // ptr<const DateTime>  — var binding but data class is immutable

// value class — var binding → mutable pointer
var pt: Point = Point.new(1, 2)
val p4 = pt.address()            // ptr<Point>

// ref — always const pointer
fun inspect(r: ref User) {
    val p5 = r.address()         // ptr<const User>
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

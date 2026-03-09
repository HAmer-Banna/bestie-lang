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
4. **No unsafe escape hatches**
5. **Minimal cognitive overhead**
6. **Uniform rules for all domains**

Bestie rejects designs where:

* Safety is optional
* Allocation is implicit
* Different subsystems use different memory rules

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

* `new()` or `allocate()` is explicitly invoked
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
* Immutable collections

Properties:

* Copied on assignment
* Safe to pass across threads
* No ownership tracking needed

---

### 4.2 Reference Types

Reference semantics exist **only** via explicit constructs:

```
own T
ref T
ptr<T>
```

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

Therefore, functions return `own` by default:

```bestie
fun createUser(): User {
    return User.new()
}
```

Conceptually:

```bestie
fun createUser(): own User
```

Rules:

* Returned objects are **owned by the caller**
* Ownership transfer is explicit
* Returning `ref` requires the owner to outlive the caller
* Returning values copies

This eliminates:

* Escaping borrows
* Hidden heap sharing
* Lifetime ambiguity

---

## 7. Deallocation: `free()` vs `freeDeep()`

### 7.1 `free()`

```bestie
student.free()
```

`free()`:

* Frees **only the object itself**
* Does **not** recurse into owned fields
* Leaves owned sub-objects intact

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

Allowed only when:

* Offset is known at compile time
* Bounds can be proven safe

```bestie
p.offset(2).val
```

Out-of-bounds access is a **compile-time error** whenever provable.

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
val l = list<int>.asArray.asImmutable
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

## 13. Concurrency Rules (Preview)

* `own` values cannot be implicitly shared
* `ref` cannot cross threads
* `ptr<T>` crossing threads is explicit and unsafe

These rules ensure:

* Data-race freedom
* Deterministic memory behavior

---

## 14. Raw Pointers — `ptr<T>`

Pointers are part of Bestie because it is a **systems language**.

They enable:

* Pass-by-address
* Explicit aliasing
* Low-level data structures
* Memory-mapped / OS / FFI interaction
* Performance-critical operations

`ptr<T>` represents **an address**, not ownership.

Rules:

* No ownership
* No lifetime guarantee
* No implicit dereference

---

### 14.1 Getting a Pointer

```bestie
val p = value.address()
```

Return type:

| Value kind      | Result         |
| --------------- | -------------- |
| Mutable value   | `ptr<T>`       |
| Immutable value | `ptr<const T>` |

---

### 14.2 Dereferencing

```bestie
val v = p.val
p.val = 42
```

No implicit dereferencing exists.

---

### 14.3 Pointer vs `ref`

| Feature       | ref         | ptr                 |
| ------------- | ----------- | ------------------- |
| Lifetime safe | Yes         | No                  |
| Escapes scope | No          | Yes                 |
| Stored        | No          | Yes                 |
| Intended use  | Safe borrow | Systems indirection |

---

### 14.4 Pass-by-Address

Pointers enable explicit indirection:

```bestie
fun fill(buf: ptr<byte>, n: int)
```

This replaces hidden reference semantics found in other languages.

---

### 14.5 Pointer and Ownership Interaction

Pointer **does not own memory**.

Modifying through pointer is allowed **only if underlying object is mutable and alive**.

```bestie
own u = User.new()
val p = u.address()
p.val.name = "Ali"   // allowed

u.free()             // must not occur while pointer used
```

Compiler validates only **local lifetime safety** to keep compilation fast.

---

## 15. FFI Pointer Rules

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

## 16. What This Model Rejects

Explicitly rejected designs:

* Garbage collection
* Implicit reference counting
* Arena sublanguages
* Unsafe blocks
* Borrow inference complexity
* Runtime lifetime tracking

---

## 17. Summary

Bestie’s memory and ownership model is:

* Explicit, not magical
* Ownership-driven, not GC-driven
* Pointer-friendly, not pointer-hiding
* Deterministic, not runtime-driven
* Safe by construction, not by convention

`own` answers **who frees**
`ref` answers **who borrows**
`ptr<T>` answers **where in memory**

Memory is not managed *for* the developer.
Memory is managed **with** the developer — explicitly, safely, and predictably.

Nothing more.
Nothing less.

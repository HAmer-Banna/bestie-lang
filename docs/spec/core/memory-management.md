# Bestie Language — Memory Management & Ownership Model

This document defines Bestie’s memory model, ownership qualifiers, pointer semantics, and deallocation rules.

Memory management in Bestie is:

* Manual
* Explicit
* Deterministic
* Compile-time validated
* Systems-grade

Bestie does **not** hide memory.
It makes memory **predictable, inspectable, and intentional**.

---

## 1. Why Bestie Has an Ownership System

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

### 1.1 The Core Problem

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

### 1.2 The Explicit Answer

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

## 2. `own` and `ref` — What They Mean

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

### 2.1 `own` — Ownership

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

### 2.2 `ref` — Borrowed Reference

`ref` means:

* This value is **not owned**
* Lifetime is controlled elsewhere
* The reference is temporary and scoped

Example:

```bestie
fun printUser(user: ref<User>) {
    print(user.name)
}
```

Rules:

* `ref` cannot outlive its owner
* `ref` cannot be stored
* `ref` cannot escape scope
* `ref` never implies ownership

---

## 3. Why Functions Return `own` by Default

When a function creates or produces a value, **someone must own it**.

Returning `ref` by default would be unsafe and misleading.

Therefore:

```bestie
fun createUser(): User {
    return User.new()
}
```

Desugars conceptually to:

```bestie
fun createUser(): own User
```

Rules:

* Returned objects are **owned by the caller**
* Ownership transfer is explicit and visible
* Lifetimes remain local and predictable

This rule eliminates:

* Escaping borrows
* Hidden heap sharing
* Lifetime ambiguity

---

## 4. Deallocation: `free()` vs `freeDeep()`

### 4.1 `free()`

```bestie
student.free()
```

`free()`:

* Frees **only the object itself**
* Does **not** recurse into owned fields
* Leaves owned sub-objects intact

This is useful for:

* Manual lifecycle control
* Custom teardown logic
* Performance-sensitive paths

---

### 4.2 `freeDeep()`

```bestie
student.freeDeep()
```

`freeDeep()`:

* Frees the object
* Recursively frees all `own` fields
* Skips all `ref` fields

This matches **real-world expectations** for structured objects and avoids:

* Writing 10 manual `free()` calls
* Boilerplate destructors
* Error-prone teardown code

The behavior of `freeDeep()` is **entirely determined at compile time**.

---

## 5. Pointer Model (`ptr<T>`)

Bestie is a systems language.
Pointers exist and are first-class.

```bestie
ptr<T>
```

means:

* A raw memory address
* No ownership semantics
* No lifetime guarantees
* Explicit dereferencing

Pointers are honest and unsafe by design.

---

## 6. Addressability Model

Bestie deliberately separates **ownership** from **addressability**.

Having an address does **not** mean ownership.
Having ownership does **not** require exposing an address.

---

### 6.1 `.address()` — The Only Way to Get a Pointer

Bestie provides **one** mechanism:

```bestie
value.address()
```

`.address()` returns a pointer to the **language-level memory representation** of a value.

Important:

> `.address()` never lies about what it points to.

---

### 6.2 Return Type of `.address()`

The type of pointer returned depends on **mutability and semantics**:

| Value kind                 | Result         |
| -------------------------- | -------------- |
| Mutable value              | `ptr<T>`       |
| Immutable or `const` value | `ptr<const T>` |
| Runtime resource           | `ptr<const T>` |

Examples:

```bestie
val x: int = 10
val px = x.address()           // ptr<const int>

var y: int = 20
val py = y.address()           // ptr<int>
```

---

## 7. Runtime Resources (Files, Sockets, etc.)

Runtime resources **are addressable**.

This is important for systems programmers.

```bestie
val f = file.open()
val p = f.address()   // ptr<const file>
```

What this means:

* The pointer refers to the **handle object**
* Not the OS kernel resource
* The handle is read-only
* Mutation through the pointer is forbidden

This is **exactly equivalent** to `FILE*` in C.

---

## 8. User-Defined Types and `.address()`

User-defined classes and structs:

* Have real layout
* Have stable memory identity
* Are fully addressable

```bestie
val own u = User.new()
val p = u.address()   // ptr<User>
```

If the binding is immutable:

```bestie
val u = User.init(...)
val p = u.address()   // ptr<const User>
```

No special casing.
No hidden rules.

---

## 9. Dereferencing Rules

Bestie has **no implicit dereferencing**.

```bestie
val v = p.val()
p.val(42)
```

Const correctness is enforced:

```bestie
ptr<const T>  // read-only
ptr<T>        // mutable
```

---

## 10. `own`, `ref`, and Pointers Together

These concepts are orthogonal:

* `own` → who frees
* `ref` → who borrows
* `ptr<T>` → raw address

You may freely combine them **where rules allow**.

What is forbidden is:

* Implicit ownership through pointers
* Lifetime extension through pointers
* Hiding ownership in pointer APIs

---

## 11. Stack vs Heap

* Value types prefer stack allocation
* Escape analysis applies
* Heap allocation is explicit (`.new()`)

The compiler may:

* Inline
* Elide
* Reorder
* Optimize

As long as observable behavior is preserved.

---

## 12. Concurrency Rules (Preview)

* `own` values cannot be implicitly shared
* `ref` cannot cross threads
* `ptr<T>` crossing threads is explicit and unsafe

These rules follow naturally from the ownership model.

---

## 13. Summary

Bestie memory management is:

* Explicit, not magical
* Pointer-friendly, not pointer-hiding
* Ownership-driven, not GC-driven
* Practical for systems
* Safe enough for backend

`own` answers **who frees**
`ref` answers **who borrows**
`ptr<T>` answers **where in memory**

Nothing more.
Nothing less.


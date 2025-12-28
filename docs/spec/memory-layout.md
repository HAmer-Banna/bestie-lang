# Memory Layout and Ownership Model

This document defines how memory is **laid out**, **owned**, **passed**, and **freed** in Bestie.

Memory behavior in Bestie is:

* Explicit
* Predictable
* Compile-time enforced
* Unified across system and backend domains

There is **one memory model**.

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

## 2. Memory Regions

Bestie recognizes three conceptual memory regions.
These are **semantic**, not exposed as language modes.

### 2.1 Stack

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

### 2.2 Heap

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

### 2.3 Static / Read-only Memory

Used for:

* Constants
* Immutable literals
* Compile-time known values

Example:

```bestie
math.PI
```

---

## 3. Value Semantics vs Reference Semantics

### 3.1 Value Types

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

### 3.2 Reference Types

Reference semantics exist **only** via explicit constructs:

```
own T
ref T
ptr<T>
```

There is no implicit reference behavior.

---

## 4. Ownership (`own`)

### 4.1 Definition

`own T` represents **exclusive ownership** of a heap-allocated object.

Rules:

* Exactly one owner at a time
* Ownership is moved, never copied
* Owner is responsible for freeing

Example:

```bestie
own user = User.new()
```

---

### 4.2 Ownership Transfer

Ownership may be transferred via `move`:

```bestie
own a = User.new()
own b = move a
```

After move:

* `a` becomes invalid
* Any use is a compile-time error

---

### 4.3 Destruction

Owners must be freed explicitly:

```bestie
user.free()
```

For deep structures:

```bestie
user.freeDeep()
```

Rules:

* `free()` frees the object itself
* `freeDeep()` recursively frees owned fields
* Compiler enforces correct destruction order

---

## 5. References (`ref`)

### 5.1 Definition

`ref T` is a **non-owning**, temporary reference.

Rules:

* Cannot outlive the owner
* Cannot cross thread boundaries
* Cannot be stored in heap objects unless immutable

Example:

```bestie
fun printUser(ref u: User): void
```

---

### 5.2 Lifetime Enforcement

The compiler ensures:

* The owner outlives all references
* No dangling references are possible
* No reference cycles exist

---

## 6. Raw Pointers (`ptr<T>`)

### 6.1 Definition

`ptr<T>` represents a **memory address**, not ownership.

Rules:

* No implicit dereference
* Must be explicitly obtained via `address()`
* Dereferencing requires `deref()`

Example:

```bestie
ptr<int> p = x.address()
val v = p.deref()
```

---

### 6.2 Pointer Arithmetic

Allowed only when:

* Offset is known at compile time
* Bounds can be proven safe

```bestie
p.offset(2).deref()
```

Out-of-bounds access is a **compile-time error** whenever provable.

---

## 7. Classes and Memory Layout

### 7.1 Inlining Rules

By default:

* `data class`, `value class`, `class` → inlined
* `open class` → inlined if possible

Inlining means:

* No hidden pointers
* Predictable field layout
* Cache-friendly memory access

---

### 7.2 Field Ownership

Fields must declare ownership intent explicitly:

```bestie
class Student {
    ref course: Course
    own address: Address
}
```

Destruction rules:

* `free()` requires owned fields to be freed first
* `freeDeep()` handles this automatically

---

## 8. Collections and Memory

### 8.1 Collection Ownership

Collections own their elements **only if element type is `own T`**.

Example:

```bestie
list<own User> users
```

* `users.freeDeep()` frees all users
* `users.free()` frees only the container

---

### 8.2 Copying Collections

* Value collections → deep copy
* Reference collections → reference copy
* Ownership collections → move only

Illegal:

```bestie
val a: list<own User>
val b = a   // compile-time error
```

---

### 8.3 Immutable & Concurrent Variants

```bestie
val l = list<int>.asArray.asImmutable
```

Rules:

* Immutable collections may be shared across threads
* No mutation APIs are available
* Compile-time enforced

---

## 9. Function Calls and Memory

### 9.1 Parameter Passing

Default: pass by value

Explicit:

* `ref` → borrow
* `own` → move

Example:

```bestie
fun process(own data: Data)
```

---

### 9.2 Return Values

* Returning `own` transfers ownership
* Returning `ref` requires owner to outlive caller
* Returning values copies

---

## 10. Scope-Based Cleanup

Bestie allows **explicit scope-based cleanup**, but never implicit GC.

Example:

```bestie
{
    own file = File.open()
    // use file
    file.free()
}
```

Leaving scope without freeing is a compile-time error unless:

* Ownership was moved
* Ownership was returned

---

## 11. No Null, No Undefined Behavior

Bestie guarantees:

* No null references
* No use-after-free
* No double free
* No uninitialized memory access

Violations are caught at **compile time**, not runtime.

---

## 12. Relation to Concurrency

Memory rules integrate directly with concurrency:

* Only immutable or owned data may cross threads
* References cannot escape thread boundaries
* Pointer usage is restricted in concurrent contexts

This ensures:

* Data-race freedom
* Deterministic memory behavior

---

## 13. What This Model Rejects

Explicitly rejected designs:

* Garbage collection
* Implicit reference counting
* Arena sublanguages
* Unsafe blocks
* Borrow inference complexity
* Runtime lifetime tracking

---

## 14. Summary

Bestie’s memory model provides:

* C-level control
* Rust-like safety (without complexity)
* Backend-friendly predictability
* Compile-time guarantees
* A single mental model

Memory is not managed *for* the developer.
Memory is managed **with** the developer — explicitly, safely, and deterministically.

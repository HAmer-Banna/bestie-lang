# Bestie Language — Memory Model & Ownership

This document defines Bestie’s memory model, ownership system, and layout guarantees.

Memory management in Bestie is:
• Manual
• Explicit
• Deterministic
• Compile-time validated
• Free of undefined behavior

Bestie does not attempt to hide memory.
It makes memory predictable, analyzable, and efficient.

⸻

## 1. Memory Philosophy

Bestie is a systems language first, backend language second.
It does **not** attempt to model every programming paradigm.

Bestie rejects:
• Garbage collection
• Implicit allocation
• Implicit sharing
• Hidden reference counting
• Runtime-only memory rules

Bestie enforces:
• Explicit ownership
• Explicit allocation and deallocation
• Compile-time lifetime reasoning
• Zero-cost abstractions
• Optimal memory layout

**Golden Rule**

If memory behavior can be resolved at compile time, it must be resolved at compile time.

⸻

## 2. Core Memory Types

Bestie defines three fundamental memory-related types:

| Type   | Meaning                            |
| ------ | ---------------------------------- |
| own<T> | Unique ownership                   |
| ref<T> | Borrowed reference                 |
| ptr<T> | Raw pointer (explicit, controlled) |

These types are orthogonal to classes, functions, and collections.

⸻

## 3. Addressability Model

Bestie deliberately separates **natural addressability** from **forced address exposure**.

### 3.1 `.address()` — Natural Address

`.address()` is a **method**, available only on types whose memory identity is meaningful.

Properties:
• Returns `ptr<T>` or `ptr<const T>`
• Never forces allocation
• Never fabricates storage
• Zero-cost

Examples:

```
val x: int = 10
val px = x.address()
```

```
val f = file.open()
f.address()    // ❌ compile-time error
```

`.address()` exists only where the address *makes sense*.

---

### 3.2 `rawAddress(value)` — Forced Address

`rawAddress(value)` is a **std-lib function**, not a method.

It may:
• Force stack or heap materialization
• Create temporary storage
• Expose implementation layout

Its use is explicit and intentional.

Examples:

```
val f = file.open()
val p = rawAddress(f)
```

Rules:
• Not allowed for abstract runtime resources (e.g. `file`)
• Not allowed for protocols
• Forbidden for values without a stable layout

`rawAddress` is a *tool of last resort*.

⸻

## 4. own<T> — Ownership

`own<T>` represents exclusive ownership of a value or allocation.

```
val own user: User = User.new()
```

Properties:
• Exactly one owner
• Owner is responsible for deallocation
• Ownership transfer is explicit
• No implicit copying

Rules:
• `own<T>` cannot be copied
• `own<T>` cannot be captured by lambdas
• `own<T>` cannot be aliased

### 4.1 own and const

```
const own user: User   // ❌ illegal
```

Reason:
• `const` implies compile-time existence
• `own` implies runtime lifetime

⸻

## 5. ref<T> — Borrowed Reference

`ref<T>` is a non-owning, temporary borrow.

```
fun printUser(user: ref<User>) {
    print(user.name)
}
```

Rules:
• Cannot outlive owner
• Cannot be stored
• Cannot escape scope

`ref<T>` never implies addressability.

⸻

## 6. ptr<T> — Raw Pointer

`ptr<T>` is a raw memory address.

```
val p: ptr<int> = x.address()
```

Properties:
• No ownership semantics
• No lifetime guarantees
• Explicit dereferencing

### 6.1 ptr and const

```
const PI: float64 = 3.14
val p = PI.address()   // ptr<const float64>
```

Mutation is forbidden through `ptr<const T>`.

⸻

## 7. Dereferencing Model

No implicit dereferencing.

```
val v = p.val()
p.val(42)
```

Mutation is always explicit.

⸻

## 8. Collections and ptr

### 8.1 `ptr<List<T>>`

Points to the collection structure.
Mutation affects the list itself.

```
val p = list.address()
```

### 8.2 `List<ptr<T>>`

List of independent pointers.
No ownership implied.

```
val l = List<ptr<int>>()
l.add(x.address())
```

These are semantically different and never interchangeable.

⸻

## 9. Strings

`str` is immutable.

```
val s: str = "hello"
val p = s.address()   // ptr<const char>
```

• No mutation allowed
• Internal layout is opaque
• `rawAddress(s)` is forbidden

⸻

## 10. Functions and Function Pointers

Functions are not values.

• No `.address()`
• No layout guarantees

Function pointers are explicit:

```
val fp: ptr<fun(int)->int> = rawAddress(myFunc)
```

This is an FFI-level feature.

⸻

## 11. Files and Runtime Resources

Files are runtime-managed resources.

```
f.address()      // ❌
rawAddress(f)    // ❌
```

Reason:
• No stable memory identity
• OS-managed lifetime
• Layout is meaningless

⸻

## 12. Allocation Rules

```
val own user = User.new()
```

• `.new()` allocates
• `init()` never allocates

Deallocation:

```
user.free()
user.freeDeep()
```

⸻

## 13. Stack vs Heap

• Value types prefer stack
• Escape analysis applies
• Heap allocation is explicit

Compiler may inline or elide allocations freely.

⸻

## 14. Memory Layout Guarantees

• Optimal padding
• Minimal headers
• No hidden metadata
• No implicit vtables

Layout is optimized, not user-defined.

⸻

## 15. Concurrency Rules

• `own<T>` cannot be implicitly shared
• `ref<T>` cannot cross threads
• `ptr<T>` crossing threads is explicit and unsafe

⸻

## 16. Summary

Bestie memory is:
• Explicit
• Honest
• Deterministic
• Systems-grade

Every address means something.
Every pointer is intentional.

⸻

Related Documents
• core.md
• oop.md
• fp.md
• concurrency.md
• errors.md

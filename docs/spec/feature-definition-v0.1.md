# Bestie — Feature Definition v0.1

---

# 1. Primitive Types

Bestie provides primitive types that map directly to native machine types:

- `int32`
- `int64`
- `uint32`
- `uint64`
- `float32`
- `float64`
- `byte`
- `bool`
- `char`
- `str` (UTF-8, immutable)

## Type inference for width
If a programmer writes:

```
val x: int
```

the compiler chooses the optimal size (`int32` or `int64`) based on target architecture.

`double` is **removed**—it is an alias for `float64`.

Everything is resolved at **compile time**.

---

# 2. Compile-Time Driven Language

Bestie resolves *every possible aspect* at compile time:

- type inference  
- generic specialization (no erasure)  
- memory layout calculation  
- padding & alignment optimization  
- protocol dispatch  
- inline expansion  
- concurrency safety (deadlocks, races, sharing rules)  
- deterministic destructor insertion  
- pointer safety  
- builder chain resolution  
- error handling paths  
- loop unrolling opportunities  
- devirtualization

The runtime is intentionally **minimal**.

---

# 3. Classes Overview

### data class  
Immutable, inlined, DTO-only.  
Zero memory overhead.

### value class  
Inlined scalar wrappers.

### class  
Final & inlined by default.  
Allocated explicitly via `.new()`.

### open class  
Normal OOP extension.

### abstract class  
Same cost as concrete class; no runtime vtables.

### single class  
Compile-time singleton.

### enum  
Numeric, inlined.

### enum class  
Named variants with data.

### protocol  
Behavior-only definition.  
Used for operators, typeclasses, iterators, builders.

### group protocol  
For bundling common behavior sets (Printable, Hashable, Equatable).

---

# 4. Memory Model Summary

Bestie uses:  
**Explicit allocation + compile-time validated safe pointers.**

```
val user = User.new()
val addr = user.address()
val u2   = addr.deref()
user.free()
```

No nulls.  
No GC.  
No unsafe blocks.  
No dangling pointers — compiler prevents them.

---

# 5. Why Bestie Is Still as Fast as C

### ✔ 1. Inlined classes by default  
No header allocation, no object wrappers, no hidden indirection.

### ✔ 2. No GC — no scanning, no barriers  
All memory cost is explicit.

### ✔ 3. No virtual dispatch unless explicitly requested  
Protocols are resolved at **compile time** through monomorphization (Rust-style).  
Zero runtime lookup.

### ✔ 4. Data layout optimized at compile time  
Given:

```bestie
class User { int x; char y; char z }
```

Compiler automatically rearranges to:

```
char y; char z; int x
```

to remove padding.

Equivalent to hand-optimized C.

### ✔ 5. No runtime type system  
Everything known at compile time.  
No reflection overhead (optional tools only).

### ✔ 6. Specialization for generics  
`List<int32>` and `List<str>` generate two different specialized machine code implementations—like C++ templates but safe.

### ✔ 7. Zero-cost error handling (Zig-style)  
No stack unwinding, no thrown exceptions.

### ✔ 8. Bound-checked containers compiled out in release mode  
`list[i]` bounds checks elided when provably safe.

### ✔ 9. Concurrency safety at compile time  
No runtime locks unless user explicitly uses them.

### ✔ 10. Deterministic destructors with zero overhead  
Compiler knows exactly when to free memory—no runtime scanning.

**These 10 points make Bestie equivalent to C in performance while offering elegance and safety.**

---

# 6. Collections System (v0.1)

### List
```
List.asArray
List.asLinked
List.asMatrix
List.asImmutable
List.asConcurrent
```

### Set
```
Set.asHash
Set.asTree
Set.asImmutable
Set.asConcurrent
```

### Map
```
Map.asHash
Map.asTree
Map.asLinked
Map.asImmutable
Map.asConcurrent
```

### Deque
```
Deque.build()
Deque.asStack()
Deque.asQueue()
```

All rely on a **Builder Protocol**.

---

# 7. Threading Model (v0.1)

Bestie includes:

- OS threads (`ThreadOs`)
- Lightweight green threads (`LightThread`)
- Actor model
- Channels (buffered/unbuffered)
- Structured concurrency API
- Compile-time safety for:
  - data races  
  - locks  
  - deadlocks  
  - unsafe sharing  
  - misuse of actors

Goal: **Rust safety + Kotlin coroutines ergonomics + C speed.**

---

# 8. Built-in Conventions

1. Naming consistency:  
   - always `add()`, not mix add/push/append  
   - always `remove()`, not delete/drop  
   - always `.count()` for size  
   - always `.length()` for strings  

2. Short keywords:  
   - `pkg`, `fun`, `ext`, `impl`, `pub`, `priv`, `pkg` (internal)

3. Formatting enforced (Go-style).

4. Built-in GOF design protocol groups (Factory, Strategy, Builder).

---

# 9. Everything in Bestie Is Designed for Decades of Stability

No feature is added unless:

- It is **fully consistent** with Bestie’s mission  
- It works for both backend and systems developers  
- It has no runtime penalty  
- It is predictable  
- It integrates cleanly with the type system  
- It will not require breaking changes in future versions

---

# 10. Versioning Philosophy

Bestie uses:

- **Semantic versioning**
- **Strict backward compatibility for stable features**
- **Compiler-level experimental flags** (so we can test features before accepting them)

---

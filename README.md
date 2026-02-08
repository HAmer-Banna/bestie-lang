# Bestie Programming Language  
**“Fast as light. Native with no GC. Control of C. Beauty of Kotlin.”**  
**Elegance with Excellence**

---

## 🌊 Vision

Bestie is a modern compiled language built to unify two worlds that historically never intersect cleanly:

- **Systems programming** (C, Zig, Rust)
- **Backend & application development** (Java, Kotlin, Go)

Bestie’s mission:

> **Bring the speed and control of system languages to backend developers,  
> and bring the expressiveness and elegance of modern OOP to systems developers.**

---

## ⚡ Core Principles

### ✔ No Garbage Collector  
Memory is always explicit.  
No GC pauses. No unpredictable runtime cost.

### ✔ Native performance  
Bestie compiles to machine code with zero runtime overhead.  
All decisions are resolved at **compile time** — types, memory layout, dispatch, concurrency safety, and generics.

### ✔ Elegant & expressive  
A clean Kotlin-inspired syntax with predictable conventions.

### ✔ Safe control  
Pointers, allocation, and freeing exist — but **fully checked** by the compiler.

---

## 🌱 Language Elements (high-level)

- `package`
- `data class` (immutable, inlined DTO)
- `value class` (inlined scalar wrapper)
- `single class` (inline singleton)
- `class` (final, inlined by default)
- `open class`
- `abstract class`
- `enum`
- `enum class`
- `protocol`
- `group protocol`
- `fun`
- `Lambdas`

Every class automatically gets:

- `.new()`
- `.free()`
- `.freeDeep()`
- `.address()`
- `.val`

---

## 🔁 Concurrency Model (summary)

- Lightweight threads
- Structured concurrency
- Actor model
- Channels
- Compile-time detection of:
  - data races  
  - deadlocks  
  - improper locking  
  - shared mutable violations

---

## 📦 Standard Library Overview

### Collections
- `List`  
- `Set`  
- `Map`  
- `Deque` (→ `asStack`, `asQueue`)  
- `Heap.asMax / asMin`  
All using the **Builder Pattern Protocol**:

```bestie
val nums = List.asArray.of(1,2,3)
val map  = Map.asHash.of({1:10, 2:20})
```

### Threading
- `ThreadOs`
- `LightThread`
- `Actor`
- `Channel`

### IO
- `File.open`, `File.close`
- `Socket`, `Tcp`, `Udp`

### DateTime
- `date.new()`
- `time.new()`

---

## 🧱 Project Structure

```
/bestie
    /compiler
    /stdlib
    /spec
        feature-definition-v0.1.md
        memory-model.md
        concurrency-model.md
    /docs
        vision.md
        design-philosophy.md
        formatting-rules.md
    /branding
        /logo
        /mascot
    README.md
```

---

## 🌊 Branching Model

Main development branch:

```
first-wave
```

Future branches:

- second-wave
- memory-core
- syntax-layer
- type-system
- concurrency-engine

---

## 📘 Full Language Specification  
For the detailed design, see:

➡️ **/spec/feature-definition-v0.1.md**

---

## 🤝 Contributing
Bestie welcomes contributions once the core specification is finalized.  
The mission: **combine power and elegance into one unified language.**

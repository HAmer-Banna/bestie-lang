# Bestie Programming Language
**Fast as light. Native. Deterministic. No GC.**
**Control of C. Elegance of Kotlin.**
**From kernel to cloud with one language**

---

## 🌊 Vision

Bestie is a modern compiled language designed to unify two domains that rarely meet cleanly:

- **Systems programming** — C, Zig, Rust
- **Backend & application engineering** — Java, Kotlin, Go

Bestie’s mission:

> **Bring the control and predictability of system languages to backend developers,
> and bring the clarity and expressiveness of modern OOP to systems developers.**

Bestie is built from first principles — not constrained by C/C++ legacy and not driven by academic ideology.

---

## ⚡ Core Principles

### ✔ No Garbage Collector
Memory is explicit and deterministic.
No pauses. No hidden allocation. No unpredictable runtime behavior.

### ✔ Native performance
Bestie compiles directly to machine code with a minimal runtime.
Most decisions are resolved at **compile time** — types, memory layout, dispatch, generics, and concurrency safety.

### ✔ Deterministic by design
If a program compiles under declared ownership/safety semantics, its behavior remains predictable over time.
No hidden costs. No runtime surprises.

### ✔ Explicit control, safe by default
Ownership-validated (`own/ref`) code paths are compiler-checked for illegal sharing and lifetime misuse.
Unsafe power (`ptr`, FFI, manual `free`) is allowed only through explicit syntax at the call site.

### ✔ Elegant and structured
A clean, Kotlin-inspired syntax with strong compile-time guarantees and a consistent mental model.

---

## 🛡️ Safety Philosophy

Bestie safety is **not about preventing malicious code**.

Bestie safety means:

> A program that stays within declared safe semantics should not silently become unstable over time.

In ownership-validated code paths, Bestie prevents:

- Forgotten frees
- Double frees
- Accidental shared mutation
- Hidden allocation
- Hidden undefined behavior

At explicit unsafe boundaries (`ptr`, FFI, manual `free`), behavior is programmer-controlled and visibly marked in source.

All without garbage collection and without runtime overhead.

---

## 🌱 Language Elements (high-level)

Bestie provides a compact but expressive core:

- `package`
- `data class` — immutable structural type
- `value class` — inline value wrapper
- `single class` — process-level singleton
- `class` — identity type (final by default)
- `open class` / `abstract class`
- `enum` / `enum class`
- `protocol` — behavior contracts (static by default)
- `fun` — compile-time resolvable functions
- `lambda` — explicit, allocation-free by default

---

## ⚙️ Performance Philosophy

Bestie guarantees **C/Zig-class performance** at the core level:

- No hidden allocation
- Static dispatch by default
- Deterministic memory layout
- Explicit concurrency
- Minimal runtime

Higher-level frameworks may trade performance for productivity, but never change the core cost model.

---

## 🧱 Architecture

Bestie is structured in strict layers:

```

core → std-lib → std-api → std-framework

```

- **Core** — language guarantees and invariants (nearly sealed)
- **Std-lib** — essential utilities and value abstractions
- **Std-api** — system and OS-level interfaces
- **Std-framework** — minimal foundation for higher-level frameworks

Each layer builds on the previous one without breaking core guarantees.

---

## ❌ What Bestie is NOT

- Not a garbage-collected language
- Not a scripting or VM-based runtime language
- Not a C++ successor or compatibility layer
- Not a framework-heavy ecosystem
- Not designed to solve every problem domain

Bestie is focused: **deterministic, native, explicit, and stable.**

---

## 🌊 Branching Model

Active development branch:

```

first-wave

```

Future architecture branches:

- `second-wave`
- `memory-core`
- `syntax-layer`
- `type-system`
- `concurrency-engine`

---

## 📘 Language Specification

Full specification and design documents:

➡️ `docs/spec/` (core language specification)

---

## 🤝 Contributing

Contributions will open once the core language stabilizes.
Bestie evolves carefully — **core stability and predictability come first.**

Mission:

> **Unify performance, control, and clarity into one deterministic native language.**

# Bestie Programming Language
**Fast as light. Native. Deterministic. No GC.**
**Control of C. Elegance of Kotlin.**
**From kernel to cloud with one language**

> Officially **`bestielang`**, **Bestie** for short — the same convention as *Golang* / *Go*.
> Use `bestielang` when searching, tagging, or naming the project; use **Bestie** when you talk about it.

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

## 💡 Why Bestie (Origin)

Bestie started from a recurring frustration: every other week there's another tutorial on *"how to do OOP in Go / Rust / Zig."*

People clearly want **clean object-oriented design *and* systems-level performance** at the same time. That combination is exactly what C++ promised — minus the elegance. So the question became simple:

> Why not a fast, native, OOP-first language that is actually pleasant to read?
> *("Have you seen lambdas in C++?")*

Bestie is that answer: the **ambition of C++** — real classes, lambdas, protocols, zero-cost abstractions, no garbage collector — rebuilt with the **discipline and ergonomics of a modern language**. Deterministic, predictable, and readable.

**The one-liner:** *Bestie is Kotlin-like syntax, designed for systems programming from day one — not bolted on afterward.*

- Kotlin developers recognize the syntax immediately.
- Systems developers get the part that matters: native code, no GC, deterministic cost, kernel-to-cloud.

Bestie chases the goals of C++ without inheriting its baggage — no undefined behavior in safe code, no hidden cost, no ceremony.

C gives control with limited safety. C++ gives power with rising complexity. Java gives safety and takes control. Rust gives safety with a high cognitive tax. Bestie exists to keep native performance, explicit memory, and a complexity budget a working engineer can hold.

OOP, FP, and procedural style are **tools**, not identities. Bestie supports all of them and enforces none of them.

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
`own` accounting is compiler-checked. Sharing the same object at a call is `ptr<T>`, marked in source. `ptr`, FFI, and manual `free` stay visible — there is no `unsafe` block.

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
- `class` — identity type (final by default)
- `open class` / `abstract class`
- `enum` — simple tags or rich tagged variants
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

Bestie is structured in layers. **The language is all of them**, not core alone:

```
core → std-lib → std-api → (optional std-framework)
```

- **Core** — language structure (syntax, types, ownership, `thread`)
- **Std-lib** — helpers (`option`, collections, `map`/`filter`, fibers)
- **Std-api** — talking to the outside (OS, files, console, HTTP, FFI)
- **Std-framework** — optional real-world stacks

If a helper in lib and a type in api would share a name, **lib wins**. Specs live under `docs/spec/` (`lang.md` is core syntax, not “the whole language”). The platform contract is `docs/platform.md`.

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

- **Landing / identity** — this README
- **Platform contract** (locked pillars, layers, versioning, build) — `docs/platform.md`
- **Language spec** — `docs/spec/` (`core/lang.md` is core syntax; lib and api are the rest of the language)

---

## 🤝 Contributing

Contributions will open once the core language stabilizes.
Bestie evolves carefully — **core stability and predictability come first.**

Mission:

> **Unify performance, control, and clarity into one deterministic native language.**

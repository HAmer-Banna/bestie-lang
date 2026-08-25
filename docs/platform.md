# Bestie Language — Platform

This document is the **language constitution**: locked compiler pillars, then the layering, versioning, and build rules that keep those pillars from being broken.

Identity lives in `README.md`. Specs live under `docs/spec/`. This file is the contract between them.

---

## Locked pillars

These pillars are **non-negotiable**. A change that violates them is a design regression, regardless of feature demand.

### 1 — Compilation speed is a first-class requirement

Fast builds are a hard constraint, not an optimization target. The compiler must stay usable in large monorepos, CI, and tight edit-compile loops. Compile times should scale linearly with code size whenever possible.

Rejected when they would blow this: whole-program global inference, Rust-style lifetime solvers, C++ template explosion, combinatorially expanding macros, hidden multi-phase resolution.

If a feature significantly degrades compilation speed, it is **rejected** — even if it is expressive.

### 2 — Compiler engineering is part of the language contract

Bestie is designed *with* the compiler. Semantics must be predictable, analyzable, and optimizable. Aggressive but safe transformations are expected.

Explicit ownership, no hidden allocation, no null, and a sealed core exist so the optimizer has a stable contract.

If a feature cannot be reliably optimized, it does **not belong in core**.

### 3 — Memory layout is a strategic advantage

Bestie guarantees predictable, cache-friendly layout: minimal headers, tight packing, reduced pointer chasing. The programmer expresses structure; the compiler chooses layout within semantic guarantees.

The compiler may reorder, inline, flatten, and compact **when observable behavior is preserved**. Layout optimization is always on, transparent, and forward-compatible.

### 4 — Machine code quality matters

Native code quality is a core engineering job: backend improvement, PGO, architecture-aware lowering, scheduling, cache-friendly codegen — without changing program semantics.

### Invariants these pillars require

- No hidden allocation in core semantics
- Static dispatch by default
- No runtime reflection
- `own` accounting at compile time; `ptr` / FFI / manual `free` remain explicit in source
- Predictable object layout
- Explicit concurrency (`thread` in core)

### What they reject

Novel syntax, paradigm purity, academic experiments, trend-driven features, and hidden runtime abstractions do **not** override these pillars.

Bestie is a deterministic, compiler-driven native language that also supports high-performance backends. It is not scripting-first, not a research language, not macro-driven, and not runtime-centered.

**Compile fast. Optimize hard. Layout that is known. Code that is good. Everything else is secondary.**

---

## Layers

Bestie is structured in layers. **The language is the layers together**, not core alone:

```

core → std-lib → std-api → (optional std-framework)

```

| Layer | Role | Change appetite |
| ----- | ---- | --------------- |
| **Core** | Language **structure**: `class`, `fun`, `if`, `own`/`ref`/`ptr`, `thread`, syntax `T ?` / `T ! E` | Almost never. A break here is a language break. |
| **Std-lib** | **Helpers** still in the language: `option`, `result`, collections, `map`/`filter`/`fold`, fibers | Conservative, but allowed to evolve |
| **Std-api** | **Talking to the outside**: OS, files, console, HTTP, FFI, MMIO | Allowed to evolve with platforms |
| **Std-framework** | Bestie in the real world (optional; third-party install is fine) | Most likely to change |

Direction of dependency is always **core → lib → api** (then optional framework). A higher layer does not redefine a lower-layer name. If a helper in lib and a type in api would collide, **lib wins**; api picks another name.

This split exists so Bestie does not repeat Java module headaches, Python 2→3, or JavaScript `==` vs `===`. Core stays small so it can stay stable. Std-lib is not "outside the language."

Each layer has **different stability, responsibility, and evolution rules**.

---

## 1. Core Language (`core`)

**Components included:**
`base.md`, `types.md`, `fp.md`, `oop.md`, `memory.md`, `constants.md`, `modules-and-packaging.md`, `exceptions.md`, `concurrency.md`, `annotations.md`.

### Properties

- Nearly sealed — changes allowed only for:
  - Critical bug fixes
  - Safety guarantees
  - Major performance-preserving improvements
- No imports required
- Keywords and core identifiers are lowercase (`int`, `float`, `fun`)
- Foundational abstractions use lowercase (`list`, `str`, `ptr`, `byte`, `thread`)
- Core is **built-in and always available**
- Defines **semantic guarantees** (memory, ownership, layout, concurrency)
- Safe-by-default semantics for ownership-validated (`own`) paths; `ref` is a stored non-owning slot, not a borrow checker
- Explicit unsafe boundaries (`ptr`, FFI, manual `free`) remain visible in source

### Stability Target

**≈ 98% sealed**

Core is the foundation and must remain stable for decades.

---

## 2. Standard Library (`std-lib`)

Provides **high-level utilities** without system dependency.

### Libraries

- `algorithms` — sorting, searching, utilities
- `allocators` — Arena, FixedBuffer, Debug
- `collections` — set, map, deque, heap
- `functional` — map, filter, fold, composition
- `math` — numeric operations, matrices, linear algebra
- `datetime` — date/time
- `format` — formatting and templates
- `concurrency` — fibers, channels, atomics, locks (built on core `thread`)
- `utilities` — `option`, `result`, StringBuilder, copy helpers
- `patterns` — protocol-based design patterns (Factory, Builder, Proxy, Iterator, Singleton via `Lazy`/`Once`)

### Properties

- Closed but not sealed
- Evolves conservatively
- Purely library-level (no OS/file/network runtime). `concurrency` may include a fiber scheduler, linked only when `fiber` is used.
- Foundational abstractions use lowercase (`option`, `result`, `set`, `map`)
- Nominal concrete types use PascalCase (`StringBuilder`, `Date`, `Command`)
- Requires `import`
- Imported via:

```

import bestie.lib.<library>

```

### Stability Target

**≈ 80% stable**

---

## 3. Standard API (`std-api`)

Provides **system-level and external interaction**.

### APIs

- `cli` — command-line
- `db` — database access
- `http` — HTTP primitives
- `io` — filesystem & streams
- `network` — sockets & protocols
- `os` — operating system
- `ext.memory` — memory inspection
- `foreign` — FFI / native interop

### Properties

- Stable but allowed to evolve
- Foundational abstractions imported from lower layers keep lowercase names
- API-defined nominal types use PascalCase
- Requires `import` and `bestie-project.toml`
- Imported via:

```

import bestie.api.<api>

```

- Version must match compiler compatibility
- No runtime framework behavior
- Performance-oriented

### Stability Target

**≈ 60% stable**

---

## 4. Standard Framework (`std-framework`)

High-level abstractions built **strictly on std-api and core**.

Frameworks provide structure — not runtime magic.

### Frameworks

- `web` — HTTP-based web (servlet-style, minimal core)
- `template` — MVC template engine
- `orm` — database abstraction
- `gui` — desktop UI
- `stream` — streaming / reactive
- `test` — testing
- `aop` — aspect utilities (compile-time only)
- `di` — dependency injection / IoC

### Properties

- Distributed remotely
- Downloaded automatically when declared in `bestie-project.toml`
- Requires compatible compiler version
- Imported via:

```

import bestie.framework.<framework>

```

- Built **on APIs, not on runtime**
- No reflection-based magic
- Optional layer

### Stability Target

**≈ 40% stable**

Frameworks are intentionally flexible and allowed to evolve.

---

## 5. Tools (`std-tools`)

Official tooling shipped with Bestie.

- `bestiec` — compiler
- `bestie test` — test runner
- `bestie fmt` — formatter
- `bestie dbg` — debugger
- `bestie build` — builder
- `bestie doc` — documentation generator
- `bestie mod` — module manager
- `bestie lint` — static analyzer
- `bestie make` — automation
- `bestie heap` — memory inspector

Tools evolve independently from the core language.

---

## 6. Versioning Scheme

```
Bestie v<lang>.<compiler>.<std-lib>.<std-api>
```

- `lang` — core language version
- `compiler` — compiler implementation
- `std-lib` — library version
- `std-api` — API version

Major version increases only when **core semantics change**.

---

## 7. Binary Model

Bestie produces **fully native binaries**.

Characteristics:

- Static linking by default
- Dead code elimination
- No runtime VM
- No hidden metadata
- Optional binary protection:
  - Symbol stripping
  - Control-flow obfuscation
  - Module encryption

---

## 8. Compiler Discovery Ladder

The compiler determines mode automatically.

### Tier 1 — Standalone Mode

Condition: No `bestie-project.toml`.

Available:

- Core language
- Std-lib only

Restrictions:

- No std-api
- No system-level access

Purpose:

- Kernels and drivers (no OS I/O)
- Freestanding experiments
- Logic that only needs core + std-lib helpers

Hello-world `println` and any talk to the OS require std-api (Tier 2).

---

### Tier 2 — Project Mode

Condition: `bestie-project.toml` present.

Enables:

- std-api
- std-framework
- Dependency resolution

Compiler validates version compatibility.

---

## 9. Project Specification

Example:

```toml
[project]
name = "order-processor"
version = "1.0.0"

[requirements]
lang = "1.0"
api = "2.4"

[std-frameworks]
web = "1.5"
test = "1.2"

[dependencies]
json_parser = "3.1.0"
postgres_driver = "0.9.0"
```

---

## 10. Dependency Resolution

### Frameworks (Compiler Managed)

* Cached locally
* Auto-downloaded
* Verified compatibility

### Community Libraries

* Managed via package manager
* DAG validated
* Linked statically

---

## 11. Build & Linking Model

* Fully native compilation
* Deterministic linking
* Static by default
* Dead code eliminated aggressively

### Build Steps

1. Detect mode (Standalone / Project)
2. Validate versions
3. Sync frameworks
4. Resolve dependencies
5. Compile & link
6. Produce native binary

---

## 12. Architectural Guarantee

Bestie enforces strict layering:

```
core → std-lib → std-api → std-framework
```

* Lower layers never depend on higher layers
* Core remains sealed
* Performance guarantees originate from core
* Higher layers cannot weaken core ownership/safety semantics
* Higher layers may trade performance for ergonomics

This separation ensures:

* Long-term stability
* Predictable performance
* Ecosystem flexibility

---

## Critical Enhancements Added

These are **important architectural clarifications**, not cosmetic:

1. Explicit **layering contract**
2. Stability targets per layer (98 / 80 / 60 / 40)
3. Clear **no-runtime-magic rule** for frameworks
4. Deterministic binary model clarified
5. Strict dependency direction defined
6. Clear distinction between **core guarantees vs framework flexibility**
7. Corrected API/framework compatibility model
8. Removed ambiguous wording about “built-in vs import”
9. Formalized discovery ladder
10. Reinforced native + deterministic positioning

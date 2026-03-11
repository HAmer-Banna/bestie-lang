# Bestie Language — Ecosystem, Versioning, and Build Workflow

This document defines the **Bestie ecosystem architecture**, including the **core language, standard libraries, APIs, frameworks, tools, versioning model, and compiler workflow**.

The objective is to maintain:

- Predictable evolution
- Deterministic performance
- Explicit unsafe boundaries
- Clear layering
- Long-term stability

Bestie is structured in strict layers:

```

core → std-lib → std-api → std-framework

```

Each layer has **different stability, responsibility, and evolution rules**.

---

## 1. Core Language (`core`)

**Components included:**
`lang.md`, `fp.md`, `oop.md`, `memory.md`, `modules-and-packaging.md`, `exceptions.md`, `concurrency.md`, `annotations.md`.

### Properties

- Nearly sealed — changes allowed only for:
  - Critical bug fixes
  - Safety guarantees
  - Major performance-preserving improvements
- No imports required
- Keywords and core identifiers are lowercase (`int`, `float`, `fun`)
- Core types use lowercase (`list`, `str`, `ptr`, `byte`)
- Core is **built-in and always available**
- Defines **semantic guarantees** (memory, ownership, layout, concurrency)
- Safe-by-default semantics for ownership-validated (`own/ref`) paths
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
- `collections` — list, set, map, deque, heap
- `functional` — map, filter, fold, composition
- `math` — numeric operations, matrices, linear algebra
- `datetime` — date/time
- `format` — formatting and templates
- `utilities` — helpers, Option, Result, StringBuilder
- `patterns` — protocol-based design patterns (Factory, Builder, Proxy, Iterator, Singleton via `Lazy`/`Once`)

### Properties

- Closed but not sealed
- Evolves conservatively
- Purely library-level (no runtime system)
- PascalCase naming for types
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
- `ext.concurrency` — advanced threading
- `ext.memory` — memory inspection
- `foreign` — FFI / native interop

### Properties

- Stable but allowed to evolve
- PascalCase naming for types
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

- Low Level development
- Kernal & drivers development
- Experiments
- Small tools
- Scripts

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

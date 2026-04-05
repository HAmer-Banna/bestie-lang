# Bestie Language — Modules, Packages, and Projects Specification

This document defines **how Bestie code is organized, compiled, exported, imported, and linked**.

Bestie uses a **strict three-layer structural model**:

1. **Source files** (`.bst`) — code
2. **Modules** (`.mod` / `module.toml`) — architecture
3. **Projects** (`bestie-project.toml`) — build units

Each layer has a **single, non-overlapping responsibility**.

---

## 1. Fundamental Principles

The Bestie module system is governed by the following rules:

1. Modules are the **unit of compilation**
2. Modules are the **unit of dependency**
3. Modules are the **primary visibility boundary**
4. Packages are **namespaces only**
5. Exports are explicit
6. Imports are explicit
7. No circular module dependencies
8. No implicit re-export
9. No runtime discovery, reflection, or scanning
10. Everything is resolved at compile time

There is **one model** for:

* Core
* Std-lib
* User code

---

## 2. Source Files (`.bst`)

### 2.1 Nature of `.bst` Files

A `.bst` file is a **pure source unit**.

Properties:

* Contains only Bestie code
* Has no implicit namespace
* Has no implicit module
* Can be compiled standalone

Example:

```bestie
fun main(): void {
    print("hello")
}
```

---

### 2.2 Standalone Compilation

A single `.bst` file may be compiled without a project or module:

```text
bestie run hello.bst
```

In this mode:

* Core language is available
* Std-lib is available
* No `std-api` is implicitly available
* No external dependencies are allowed

This supports:

* Scripts
* Experiments
* REPL usage
* Serverless handlers
* One-off utilities

---

## 3. Projects

### 3.1 Purpose

A project defines the **build scope**.

A project answers:

* What is built?
* For which targets?
* With which dependencies?
* With which optimizations?

A project does **not** define:

* Architecture
* Visibility
* Namespaces

---

### 3.2 Project Descriptor

Projects are defined by `bestie-project.toml`.

Example:

```toml
[project]
name = "order-service"
version = "1.0.0"

[targets]
type = "executable"

[build]
opt = "release"
lto = true

[dependencies]
std-lib = "1.0"
std-api = "1.0"
```

Rules:

* A project may contain multiple modules
* A module may belong to exactly one project

---

## 4. Modules

### 4.1 Definition

A **module** is the fundamental unit of:

* Compilation
* Dependency management
* Visibility enforcement

A module:

* Owns a set of source files
* Defines a logical namespace root
* Explicitly declares dependencies
* Explicitly declares exports

---

### 4.2 Module Descriptor

Modules are defined using a module descriptor (`.mod` or `module.toml`).
The file format is tooling-defined; **the semantics are fixed**.

Canonical example:

```toml
name = "orders"
version = "1.0.0"

sources = [
  "src/orders/**",
  "src/shared/money.bst"
]

depends = [
  "std.text",
  "payments"
]

exports = [
  "Order",
  "OrderService"
]
```

Rules:

* Module names are unique within a project
* Dependencies must form a DAG
* Transitive dependencies are allowed but never re-exported

---

### 4.3 Source Ownership

Rules:

* A source file belongs to exactly one module
* A module may include multiple directories
* A module may include individual files
* Physical layout is decoupled from architecture

---

## 5. Packages

### 5.1 Definition

A **package** is a **namespace inside a module**.

Packages:

* Organize names
* Do not define visibility
* Do not affect compilation
* Never cross module boundaries

Declared at the top of a file:

```bestie
package net.http.client
```

---

### 5.2 Package Rules

Rules:

* Folder structure must match package structure
* Packages never imply visibility
* Packages cannot be imported across modules without dependency
* Packages do not participate in export logic

---

## 6. Visibility and Encapsulation

### 6.1 Visibility Modifiers

Bestie supports four visibility levels:

| Modifier | Scope                             |
| -------- | --------------------------------- |
| `pub`    | Visible outside the module        |
| `pkg`    | Visible inside the package        |
| `protec` | Visible to subclasses             |
| `priv`   | Visible inside the declaring type |

---

### 6.2 Module Boundary Rule

* The module boundary is the **primary visibility boundary**
* Only `pub` symbols may be exported
* `pub` symbols not listed in `exports` are compile-time errors

Nothing is exported accidentally.

---

## 7. Export Rules

### 7.1 Explicit Export

A symbol is exported if and only if:

1. It is declared `pub`
2. It belongs to a module
3. It is listed in the module’s `exports`

Example:

```bestie
pub fun parseUrl(input: str): Url { ... }

pub data class Url(
    val scheme: str,
    val host: str
)
```

---

### 7.2 Non-Exportable Symbols

The following can never be exported:

* `priv` members
* `pkg` members
* Local classes
* Local functions
* Lambdas
* Non-`pub` inner classes

---

## 8. Imports and Dependencies

### 8.1 Dependency Declaration

Modules declare dependencies in their module descriptor.

If a module declares:

```toml
depends = ["net"]
```

Then code inside the module may import exported symbols from `net`.

---

### 8.2 Import Syntax

Imports are **file-scoped** and reference **exported symbols**:

```bestie
import net.http.client.HttpClient
```

Rules:

* No relative imports
* No file-path imports
* No wildcard imports (`*`)
* Imports must be explicit and fully qualified
* No package aliasing
* Symbol aliasing is allowed only for imported types/functions

---

### 8.3 Explicitness Rule

Bestie does not allow:

* Implicit imports
* Default imports (except core primitives)
* Importing undeclared dependencies

---

## 9. Default Imports

The following namespaces are available through the language prelude
(not via user-written wildcard imports):

```
core.lang
core.types
```

This includes:

* `int`, `float`, `str`
* `ptr`, `option`
* Basic annotations

Everything else must be imported explicitly.

---

## 10. Re-exporting (Forbidden)

Modules may not re-export their dependencies.

If:

* `app` depends on `net`
* `net` depends on `core`

Then:

* `app` cannot access `core` unless it declares it

This prevents:

* Dependency leakage
* Hidden coupling
* API instability

---

## 11. Cycles (Forbidden)

Module dependency cycles are compile-time errors.

```
A -> B -> C -> A   ❌
```

Cycles break:

* Compilation order
* Memory layout guarantees
* Optimization assumptions

---

## 12. Compilation Model

Bestie uses **Ahead-Of-Time (AOT) compilation**.

High-level pipeline:

1. Parse `.bst` files
2. Resolve modules
3. Build dependency graph
4. Perform semantic analysis
5. Generate IR
6. Optimize
7. Emit native artifacts

There is:

* No bytecode VM
* No runtime class loader
* No runtime symbol discovery

---

## 13. Linking Model

Bestie uses a **static linking model**.

Properties:

* Modules compile to object units
* Linking is deterministic
* Dead code elimination is aggressive
* Unused exports are removed

This enables:

* Small binaries
* Fast startup
* Serverless suitability

---

## 14. Metadata and Reflection

Rules:

* No runtime metadata tables
* No reflection
* No annotation-driven runtime behavior

Optional compile-time metadata:

* Tooling-only
* Stripped from release builds

---

## 15. REPL and Interactive Mode

REPL operates in a **virtual module**:

* No exports
* No imports except core and std-lib
* Incremental compilation

REPL semantics match the language model.

---

## 16. Design Rationale

This system ensures:

* Predictable builds
* Fast compilation
* Stable APIs
* No classpath hell
* No runtime surprises
* Native-first ergonomics

---

## 17. Summary

* Source files are **code**
* Modules are **architecture**
* Packages are **namespaces**
* Projects are **build units**

Bestie’s module system is:

* Explicit
* Deterministic
* AOT-friendly
* Architecture-enforcing
* Free of runtime magic

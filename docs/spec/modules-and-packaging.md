# Modules and Packaging

This document defines **how code is organized, packaged, compiled, and linked** in Bestie.

Bestie uses a **three-layer structure**:

1. **Source files** (`.bst`)
2. **Modules** (`.mod`)
3. **Projects** (`bestie-project.toml`)

Each layer has a **single, non-overlapping responsibility**.

---

## 1. Design Goals

The module and packaging system is designed to:

1. Scale from scripts to large distributed systems
2. Support AOT compilation and static linking
3. Enforce architectural boundaries at compile time
4. Avoid implicit behavior
5. Remain tooling-friendly
6. Avoid runtime class loaders or reflection
7. Work for:

   * System programming
   * Backend services
   * Microservices
   * Serverless
   * Games
   * Embedded software

---

## 2. Source Files (`.bst`)

### 2.1 Nature of `.bst` Files

A `.bst` file is a **pure source unit**.

Properties:

* Contains only Bestie code
* Has no package declaration
* Has no implicit namespace
* Can be compiled standalone

Example:

```bestie
fun main(): void {
    print("hello")
}
```

This enables:

* Experiments
* Scripts
* Serverless handlers
* REPL usage
* One-off utilities

---

### 2.2 Standalone Compilation

A single `.bst` file may be compiled and executed without any project or module:

```text
bestie run hello.bst
```

In this mode:

* Core language is fully available
* `std-lib` is available
* No `std-api` is implicitly available
* No external dependencies are allowed

---

## 3. Modules (`.mod`)

### 3.1 Definition

A **module** is the fundamental **unit of encapsulation and visibility** in Bestie.

A module:

* Defines a logical namespace
* Owns a set of source files
* Explicitly declares exports
* Explicitly declares dependencies

Modules are defined using `.mod` files.

---

### 3.2 Module File Structure

Example:

```bestie
module orders

sources:
    src/orders/**
    src/shared/money.bst

export:
    Order
    OrderService

requires:
    std.text
    payments
```

Sections are **explicit and ordered**.

---

### 3.3 Sources Mapping

Rules:

* A module may include **one or more directories**
* A module may include **individual files**
* Source ownership is explicit
* A source file may belong to **exactly one module**

This decouples:

* Physical layout
* Logical architecture

---

## 4. Visibility and Encapsulation

### 4.1 Visibility Modifiers

Bestie supports:

```
pub     // visible outside module
pkg     // visible within module
protec  // visible to subclasses
priv    // visible within type
```

Rules:

* Module boundary is the **primary visibility boundary**
* `pub` symbols must be listed in `export`
* Unexported `pub` symbols are a compile-time error

---

### 4.2 Module API Surface

The module API is:

* Explicit
* Auditable
* Stable by design

Nothing is exported accidentally.

---

## 5. Imports and Dependencies

### 5.1 Import Rules

Imports reference **modules**, not files.

Example:

```bestie
import orders.Order
```

Rules:

* No relative imports
* No file-path imports
* All imports must resolve to an exported symbol

---

### 5.2 Dependency Graph

Rules:

* Dependency graph must be acyclic
* Cycles are compile-time errors
* Dependencies are resolved at compile time

This enables:

* AOT compilation
* Static linking
* Fast builds

---

## 6. Project Definition (`bestie-project.toml`)

### 6.1 Purpose

`bestie-project.toml` defines the **build unit**.

It answers:

* What is built?
* For which targets?
* With which dependencies?
* With what optimizations?

It does **not** define code structure.

---

### 6.2 Example

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

---

## 7. Relationship Between Project and Modules

Rules:

1. A project may contain **multiple modules**
2. A module may not span multiple projects
3. Projects define *build scope*
4. Modules define *architectural boundaries*

This cleanly separates:

* Build concerns
* Code concerns

---

## 8. Compilation Model (High Level)

Bestie uses **Ahead-Of-Time (AOT) compilation**.

Pipeline:

1. Parse `.bst` files
2. Resolve modules
3. Build dependency graph
4. Perform semantic analysis
5. Generate intermediate representation (IR)
6. Optimize
7. Emit native binaries or artifacts

There is:

* No runtime class loader
* No bytecode VM
* No runtime symbol discovery

---

## 9. Linking Model

Bestie uses a **static linker model**, similar to C, but with higher-level semantics.

Properties:

* Modules compile to object units
* Linking is deterministic
* Dead code elimination is aggressive
* No unused exports are retained

This enables:

* Small binaries
* Fast startup
* Serverless suitability

---

## 10. Metadata and Reflection

Rules:

* No runtime metadata tables
* No class metadata area (unlike Java)
* No runtime reflection

Optional compile-time metadata:

* Used only for tooling
* Stripped from release builds

---

## 11. REPL and Interactive Mode

REPL operates in a **virtual module**:

* No exports
* No imports except core and std-lib
* Code is compiled incrementally

This keeps REPL semantics aligned with the language.

---

## 12. Distributed and Microservice Suitability

This model supports:

* Independent deployable units
* Clear ownership boundaries
* Minimal binaries
* Predictable startup times
* Explicit dependency graphs

No hidden runtime behavior exists.

---

## 13. What This System Explicitly Rejects

Bestie does not support:

* Runtime class loaders
* Dynamic module loading
* Implicit dependency resolution
* Reflection-based wiring
* Package scanning
* Annotation-driven runtime behavior

---

## 14. Summary

Bestie’s module and packaging system is:

* Explicit
* Deterministic
* AOT-friendly
* Scalable
* Architecture-enforcing

Source files are **code**.
Modules are **architecture**.
Projects are **build units**.

This separation is foundational to Bestie’s long-term stability.

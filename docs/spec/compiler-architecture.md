# Compiler Architecture

This document describes the **high-level architecture** of the Bestie compiler.

The compiler is designed to be:

* **Fast**
* **Deterministic**
* **Modular**
* **Optimized by construction**
* **Friendly to tooling and contributors**

Bestie’s compiler architecture reflects the same philosophy as the language itself:
**explicit, predictable, and engineered for the long term**.

---

## 1. Design Goals

The compiler exists to satisfy the following non-negotiable goals:

1. **Native-quality performance**
2. **Fast compilation times**
3. **Compile-time enforcement of safety rules**
4. **Whole-program optimization**
5. **Incremental and scalable builds**
6. **Clear separation of concerns**
7. **Tooling-first design**

The compiler is not a monolith.
It is a **pipeline of well-defined stages**, each with a narrow responsibility.

---

## 2. Compilation Model Overview

Bestie is an **Ahead-of-Time (AOT) compiled language**.

There is:

* No bytecode
* No virtual machine
* No runtime class loading
* No reflection-driven execution

### High-Level Pipeline

```
.bst source files
      ↓
Frontend (syntax + semantics)
      ↓
HIR (High-Level IR)
      ↓
MIR (Memory-Aware IR)
      ↓
LIR (Lowered IR)
      ↓
Object files (.o)
      ↓
Bestie-aware linker
      ↓
Native executable / library
```

Every stage is deterministic and reproducible.

---

## 3. Frontend

The frontend is responsible for **language correctness**, not optimization.

### 3.1 Responsibilities

The frontend performs:

* Lexing and parsing
* AST construction
* Name resolution
* Type checking
* Ownership and lifetime validation
* Visibility and module boundary enforcement
* Protocol conformance checks
* Compile-time annotation processing

All **language errors** are detected here.

If code reaches IR generation, it is already:

* Memory-safe
* Ownership-correct
* Race-free by construction

---

## 4. Intermediate Representations (IR)

Bestie uses **multiple IR layers**, each with a specific purpose.

This avoids the complexity and opacity found in monolithic IR designs.

---

### 4.1 HIR — High-Level IR

HIR is close to the source language.

Characteristics:

* Explicit types
* Explicit ownership (`own`, `ref`, `ptr`)
* Fully resolved generics
* No syntactic sugar
* Clear control flow

HIR preserves **developer intent** and is ideal for:

* Semantic validation
* High-level optimizations
* Tooling (linters, formatters)

---

### 4.2 MIR — Memory-Aware IR

MIR is where Bestie’s core strength appears.

MIR makes:

* Allocation
* Deallocation
* Ownership transfer
* Lifetime boundaries

**explicit and verifiable**.

Responsibilities:

* Stack vs heap placement
* Move semantics
* Destructor ordering
* Escape analysis
* Thread-boundary enforcement

If MIR is correct, memory safety is guaranteed.

---

### 4.3 LIR — Lowered IR

LIR is close to machine-level execution.

Characteristics:

* Concrete data layouts
* Resolved dispatch (static or virtual)
* Explicit calling conventions
* Platform-aware lowering

LIR is designed to map cleanly to:

* LLVM
* Or a future Bestie-native backend

---

## 5. Optimization Strategy

Bestie favors **predictable optimizations** over speculative ones.

### 5.1 Compile-Time Resolution First

Preferred optimizations:

* Inlining (guided by annotations and heuristics)
* Monomorphization
* Dead code elimination
* Escape elimination
* Constant folding
* Loop unrolling (explicit or provable)

Avoided:

* Runtime speculation
* Profile-guided surprises
* Hidden allocations

---

### 5.2 Whole-Program Optimization

Because Bestie:

* Uses static linking
* Has no dynamic class loading
* Has explicit module boundaries

The compiler can perform:

* Cross-module inlining
* Global dead code elimination
* Data layout optimization

---

## 6. Linking Model

Bestie uses a **static, language-aware linker**.

### Key Characteristics

* Symbol resolution at build time
* Explicit exports via `bestie.mod`
* No runtime symbol discovery
* Stable ABI boundaries

The linker understands:

* Ownership contracts
* Visibility rules
* Protocol dispatch tables
* Open vs closed class layouts

This is **not** a raw C linker pipeline.

---

## 7. Metadata Handling

Bestie generates metadata for **compile-time and tooling use only**.

### Metadata Includes:

* Type layouts
* Method tables
* Protocol implementations
* Ownership rules
* Annotations
* Source mappings

### Metadata Is:

* Stored in object files
* Available to the linker
* Used by debuggers and IDEs

### Metadata Is NOT:

* Accessible at runtime
* Used for reflection
* Used for dynamic dispatch decisions

---

## 8. Runtime Architecture

Bestie has a **minimal runtime**.

### Runtime Responsibilities:

* OS thread startup
* Light-thread scheduling
* Channel coordination
* Panic handling (should be unreachable in correct code)
* Foreign boundary glue

### Runtime Does NOT:

* Manage memory
* Track objects
* Load code dynamically
* Perform GC
* Interpret metadata

The runtime exists to support execution — not to control it.

---

## 9. Incremental Compilation

The compiler is designed for **fast iteration**.

Features:

* Module-level caching
* IR reuse
* Dependency-aware recompilation
* Stable hashing of inputs

A small change should never trigger a full rebuild.

---

## 10. Tooling Integration

The compiler exposes **structured internal APIs** for tools.

Supported tools:

* Formatter
* Linter
* Language server (LSP)
* Debugger
* Static analyzers

Tools operate on:

* AST
* HIR
* Metadata

Not on fragile text parsing.

---

## 11. Foreign Code Integration

Foreign code is handled outside the core language.

* Implemented via `std-api.foreign`
* Explicit ABI boundaries
* Explicit ownership contracts
* No `unsafe` keyword in the language

Foreign code is **contained**, not viral.

---

## 12. Versioning and Stability

Compiler components are versioned alongside:

```
bestie <lang>.<core>.<std-lib>.<std-api>
```

Rules:

* Compiler upgrades preserve source compatibility when possible
* IR formats may evolve internally
* Public behavior is stable within major versions

---

## 13. Summary

The Bestie compiler is:

* Modular
* Predictable
* Optimized by design
* Friendly to contributors
* Built for decades, not trends

It treats:

* Performance as a feature
* Safety as a requirement
* Tooling as a first-class concern

The compiler is not an implementation detail —
it is **part of Bestie’s identity**.

# Bestie Language — Ecosystem, Versioning, and Build Workflow

This document outlines the **Bestie language ecosystem**, including the **core language, standard libraries, APIs, frameworks, tools, versioning**, and **compiler workflow**. The goal is to provide clarity and guidance for developers, contributors, and tool authors, while maintaining **predictable evolution, high performance, and industrial-grade reliability**.

---

## 1. Core Language (`core`)

* **Components included:** `lang.md`, `fp.md`, `oop.md`, `memory.md`, `modules.md`, `collections.md`, `error.md`, `concurrency.md`, `annotations.md`.
* **Properties:**

  * Nearly sealed — open only for critical bug fixes or security evolution or extremely benifit feature that don't hurts performance
  * Requires no imports for fundamental language features
  * All keywords and identifiers are lowercase (`int`, `float`, `fun`)
  * Classes in core are lowercase (like `list`, `str`, `byte`, `ptr`)
 
  core is built in, it doesn't even needs importing

---

## 2. Standard Library (`std-lib`)

High-level, convenient utilities:

* `algorithms` — sorting, searching, and general algorithmic helpers
* `functional` — map, filter, fold, functional constructs
* `math` — numerical operations, special matrices, linear algebra
* `datetime` — date, time, and interval handling
* `format` — string formatting, templating, I/O formatting
* `utilities` — general-purpose helpers, conversions, validation

**Properties:**

* Closed but not sealed, evolves carefully
* PascalCase for classes/protocols

std-lib is built in, but it needs importing under bestie.lib.<libraryname>

---

## 3. Standard API (`std-api`)

System-level and external APIs:

* `cli` — command-line interfaces and argument parsing
* `db` — database abstractions and query helpers
* `http` — HTTP clients and server primitives
* `io` — file system, streams, and buffers
* `network` — low-level networking and protocols
* `os` — OS services, environment, and process management
* `ext.concurrency` — advanced threading and synchronization
* `ext.memory` — memory introspection and control
* `foreign` — interoperability with external libraries

**Properties:**

* Less restricted, stable for production
* Declared in `bestie-project.toml`
* Lives in `bestie.api` domain
* High-performance and safe

std-api is built in, but it needs creating of bestie-project.toml importing under bestie.api.<apiname>, must be compatable with compiler version else error

---

## 4. Standard Framework (`std-framework`)

High-level abstractions over APIs:

* `web` — high-level web abstractions over HTTP (build above http & network api)
* `template` - for MVC templates
* `orm` — database abstraction layer (build above api db)
* `gui` — desktop GUI abstractions
* `stream` — streaming and reactive utilities (build above core & api concurrency)
* `test` — testing utilities
* `aop` - Aspect Oriented Programming (In bestie can be achieved using classes/functions)
* `di` — dependency-injection, IoC and service wiring (used with other frameworks)

**Distribution:** Shipped via **remote repository**, automatically downloaded by the compiler once defined in bestie-project.toml, it needs internet connection, lives under bestie.framework.<frameworkname> must be compatable with compiler version else error

---

## 5. Tools (`std-tools`)

* `bestiec` — compiler
* `bestie test` — test runner
* `bestie fmt` — code formatter
* `bestie dbg` — debugger
* `bestie build` — project builder
* `bestie doc` — documentation generator
* `bestie mod` — module manager
* `bestie lint` — static analyzer
* `bestie make` — automation (like Make/CMake)
* `bestie heap` — memory inspection tool

---

## 6. Versioning Scheme

```
Bestie v<lang>.<compiler>.<std-lib>.<std-api>
```

* **lang** — core language version
* **compiler** — compiler implementation version
* **std-lib** — standard library version
* **std-api** — system API version

**Example:** `Bestie v1.1.2.3` → `v2` when **all four components** undergo major change.

---

## 7. Binary Protection

* Produces **optimized object files/binaries**
* Techniques for protection:

  * Strip debug symbols
  * Control-flow obfuscation
  * Inlining and code flattening
  * Optional module encryption

---

## 8. Compiler Discovery Ladder & Build Workflow

The Bestie compiler (`bestiec`) determines the available feature set using a strict discovery hierarchy.

### 8.1 Tier 1: Standalone Mode (The Script)

* **Condition:** No `bestie-project.toml` exists in the current or parent directories
* **Scope:** Only `core.lang` and `core.types` are available
* **Imports:** Only `std-lib` modules
* **Restriction:** System-level APIs (Networking, File IO, etc.) are **forbidden** for safety

### 8.2 Tier 2: Project Mode (The System)

* **Condition:** `bestie-project.toml` is present
* **Scope:** Unlocks `std-api` and `std-framework` modules defined in the TOML
* **Validation:** Compiler verifies that the project's requested `api` version is compatible with the installed `bestiec` version

---

### 8.3 Project Specification (`bestie-project.toml`)

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

### 8.4 Dependency Resolution Strategy

**Standard Frameworks (Compiler-Managed):**

* Checked in local cache, auto-downloaded if missing, integrated into build pipeline

**Community Libraries (BSTPM-Managed):**

* `bpm install` resolves DAG, verifies signatures, populates cache, and compiler links binaries

---

### 8.5 Linking & Build Summary

* **Binary-Only:** Remote repo distributes optimized object files
* **Static Linking:** Single, self-contained executable
* **Dead Code Elimination:** Strips unused code from std-lib and std-api

**Build Summary:**

1. Check for TOML → Script or Project mode
2. Verify Version → satisfy `api` and `lang` requirements
3. Sync Frameworks → auto-download missing std-frameworks
4. Verify Dependencies → BSTPM-managed modules satisfied
5. Compile & Link → produce **native, optimized binary**

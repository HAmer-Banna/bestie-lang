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
name = "acme/order-service"
version = "1.0.0"
bestie = "1.2"                 # toolchain: language, compiler, std-lib, std-api

[[target]]
name = "order-service"
kind = "executable"

[dependencies]
"acme/money" = "2.1"
"bestie/web" = "1.5"
```

Rules:

* A project may contain multiple modules
* A module may belong to exactly one project
* A project is also the unit that is **versioned and published**. Bestie calls it a *project*, never a *package*, because "package" already means a namespace (§5)
* `std-lib` and `std-api` are never listed as dependencies — the `bestie` toolchain version selects them

The full manifest format, dependency resolution, and the `bestie` tool are specified in `std-tools/package-manager.md`.

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
* Packages are **namespaces only** — they never gate or grant visibility
* Visibility is determined by the module boundary, not the package
* Two files in different packages within the same module can access each other's `internal` members
* Packages cannot be imported across modules without a declared dependency
* Packages do not participate in export logic

---

## 6. Visibility and Encapsulation

### 6.1 Visibility Modifiers

Bestie supports four visibility levels:

| Modifier | Scope                                              |
| -------- | -------------------------------------------------- |
| `public`    | Visible outside the module (exported API)          |
| `internal`    | Visible anywhere within the same module (default)  |
| `protected` | Visible to subclasses only                         |
| `private`   | Visible inside the declaring type — or, at top level, inside the declaring **file** |

`internal` is the default visibility when no modifier is written. Any file within the same module can access an `internal` symbol, regardless of which package it belongs to — packages are namespaces only and never gate visibility.

A top-level `private` declaration is **file-private**: visible only within its own `.bst` file, never exported, and unable to collide with a same-named `private` symbol in another file. See `core/oop.md` §7.

---

### 6.2 Module Boundary Rule

* The module boundary is the **primary and only visibility boundary**
* `public` — crosses the module boundary (exported API)
* `internal` — stays within the module boundary (internal API)
* `protected` and `private` — stay within the type hierarchy
* Only `public` symbols may be exported
* `public` symbols not listed in the module `exports` are compile-time errors

Nothing is exported accidentally.

---

## 7. Export Rules

### 7.1 Explicit Export

A symbol is exported if and only if:

1. It is declared `public`
2. It belongs to a module
3. It is listed in the module’s `exports`

Example:

```bestie
public fun parseUrl(input: str): Url { ... }

public data class Url {
    scheme: str
    host: str
}
```

Fields are declared in the body; the compiler generates the memberwise initializer (`oop.md` §11.4). There is no header-constructor form.

---

### 7.2 Non-Exportable Symbols

The following can never be exported:

* `private` members, including file-private top-level declarations
* `internal` members
* Local classes
* Local functions
* Lambdas
* Non-`public` inner classes

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

### 8.2 Import Declarations

Imports are **file-scoped**. Every `import` is an explicit declaration: the file may use only what it declares (plus the prelude in §9).

Three declaration forms are allowed. All of them require a declared module dependency (§8.1). All of them import only `public` symbols listed in the providing module’s `exports`. None of them re-export.

Wildcard imports (`import pkg.*`) are **not allowed**. They hide where a name comes from, collide silently across packages, and are unused anywhere in the language surface. Module and package declarations already cover “bring this API in” without `*`.

**Symbol import** — one exported symbol, available by its unqualified name:

```bestie
import net.http.client.HttpClient
import bestie.api.io.println
```

**Module import declaration** — every symbol in a **module’s** `exports`, each available by its unqualified name:

```bestie
import bestie.lib.math
import payments
```

This is the form used for std-lib, std-api, and std-framework modules:

```bestie
import bestie.lib.<library>
import bestie.api.<api>
import bestie.framework.<framework>
```

**Package import declaration** — the path names a package that is not itself a module. The last path segment is bound as a qualifier; members stay qualified:

```bestie
import bestie.lib.format.json

val user = try json.parse<User>(input)
```

A package import does not dump names into file scope and does not recurse into sub-packages.

---

### 8.3 Import Rules

* No relative imports
* No file-path imports
* No wildcard imports (`*`)
* Paths are fully qualified from a module or package root
* No package aliasing (`import net.http.client as http` is illegal)
* Symbol aliasing is allowed only for imported types and functions (`import net.http.client.HttpClient as Client`)
* Module imports cannot be aliased
* If two import declarations introduce the same unqualified name into a file, it is a compile-time error — disambiguate with a symbol import and an alias, or with a package import
* Importing a path that is not an exported symbol, a package, or a depended-upon module is a compile-time error

Resolution of an import path:

1. An exported symbol of a depended-upon module → symbol import
2. A depended-upon module’s canonical name → module import declaration
3. A package inside a depended-upon module → package import declaration
4. Otherwise → compile-time error

---

### 8.4 Explicitness Rule

Bestie does not allow:

* Implicit imports
* Default imports (except the prelude in §9)
* Importing undeclared dependencies
* Wildcard imports

Module and package imports are still explicit: the file names the module or package it uses. They do not make imports implicit, and they do not weaken the export or dependency rules.

---

## 9. Default Imports

The following namespaces are available through the language prelude.
They are not user-written imports; writing `import core.lang` is not required and is not how the prelude is provided:

```
core.lang
core.types
```

This includes:

* `int`, `float`, `str`
* `ptr`, `array`, `slice`, `range`
* Basic annotations

Everything else must be imported with an import declaration.

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

## 13. Libraries, Artifacts, and Linking

### 13.1 What a Project Can Produce

A project declares one or more **targets** (`std-tools/package-manager.md` §4.2). Each target is one of four kinds:

| Kind | Produces | Consumed by | Interface |
| ---- | -------- | ----------- | --------- |
| `executable` | A native program | The OS | `main` |
| `library` | Nothing on its own — it is **source** plus its module descriptors | Other Bestie projects, which compile it | The module's `exports` (§7) |
| `staticlib` | `.a` / `.lib` | C, C++, Rust, Zig, any C-ABI linker | `export(c)` functions only (`std-api/foreign.md` §4.3) |
| `dynlib` | `.so` / `.dylib` / `.dll` | The same, at load time — or `openLibrary` at run time | `export(c)` functions only |

Bestie code reaches other Bestie code as a **`library`**: published and consumed as source. `staticlib` and `dynlib` exist to give **other languages** a Bestie implementation behind a C interface.

### 13.2 Why There Is No Binary Bestie Library (No "Jar")

A Java `.jar` works because bytecode is portable and the JVM links it at run time. Bestie has neither, and three of its core guarantees make a precompiled Bestie-to-Bestie binary format impossible to keep stable:

* **Generics are monomorphized** (`compiler/compiler-architecture.md`). `list<T>` from a library is compiled once for each `T` the *consumer* uses — the library cannot ship code for types it has never seen.
* **Layout is not stable across compiler versions** (`memory-layout.md` §15). A `value class` compiled by one compiler may have different field offsets under the next; a binary that baked in the old offsets would read the wrong bytes.
* **Inlining and dead-code elimination cross module boundaries.** That is where "C performance" comes from, and it requires the source of both sides at once.

C++ shows what happens when a language ships binaries anyway: ABI breaks between compiler versions, and libraries pinned to one standard-library build. Bestie avoids it the way Rust, Go, and Zig do — **distribute source, cache compiled output locally**:

* Every dependency is compiled by the consumer's toolchain, for the consumer's target, with the consumer's optimization settings.
* Compiled modules are cached per (toolchain version, target, profile, source hash), so a dependency is compiled once per machine, not once per build.
* The cache is local and disposable. It is never published and never treated as an interface.

The one stable binary interface Bestie offers is the C ABI, through `export(c)`. A team that must ship a closed-source component ships a `staticlib` or `dynlib` with a C header, exactly as it would from C.

### 13.3 Linking

**Bestie code is always linked statically.** Every module of the program and of its dependencies is compiled and linked into one artifact; nothing Bestie-written is loaded at run time.

Properties:

* Linking is deterministic — the same sources, lockfile, toolchain, target, and profile produce a bit-identical artifact
* Dead code elimination is aggressive; unused exports are removed
* No Bestie runtime library needs to be installed on the machine that runs the program

**System libraries follow the platform.** The C library and OS interfaces are linked the way the platform requires:

| Target | Default | Fully static option |
| ------ | ------- | ------------------- |
| Linux | glibc, dynamic | `static = true` links musl statically — one file with no runtime dependencies |
| macOS | `libSystem`, dynamic (the only supported form) | not available |
| Windows | system DLLs (`kernel32`, `ucrt`), dynamic | static CRT |
| Freestanding | none | n/a |

**Native C libraries** used through `foreign fun` are linked as the project declares in `[native]` (`std-api/foreign.md` §11.1) — statically or dynamically, chosen per library and never implied.

**Loading code at run time** is never implicit. A program that needs plugins opens a `dynlib` explicitly with `openLibrary` (`std-api/foreign.md` §11.2) and calls its `export(c)` functions.

This enables:

* Small binaries
* Fast startup
* Serverless suitability
* One-file deployment where the platform allows it

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

# std-tools

## 1. Purpose and Scope

`std-tools` defines the **official developer tooling** shipped with the Bestie compiler distribution.

These tools support the **development lifecycle** (build, test, debug, format, document, automate) but **do not participate in program semantics or runtime behavior**.

Key principles:

- Tools are explicitly invoked
- Tools are not implicitly executed by the compiler
- Tools are orthogonal to the language core
- Tools may evolve independently from the language

---

## 2. Relationship to Bestie Layers

| Layer | Responsibility |
|------|----------------|
| core | Language semantics, syntax, safety guarantees |
| std-lib | Fundamental data structures and utilities |
| std-api | System-level APIs (IO, OS, Network, etc.) |
| std-framework | High-level frameworks (web, orm, gui, etc.) |
| std-tools | Developer tooling |

Notes:

- `std-tools` do not add language features
- `std-tools` do not affect runtime behavior
- `std-tools` do not introduce runtime dependencies

---

## 3. Distribution and Availability

### 3.1 Shipping Model

- All `std-tools` are shipped with the Bestie compiler distribution
- Shipping does not imply usage
- Tools are inactive unless explicitly invoked

### 3.2 Import Model

- `std-tools` are never imported into `.bst` files
- Tools are accessed exclusively via CLI commands

---

## 4. Tool Entry Points

Two executables are provided:

bestiec # compiler
bestie # tooling entry point

---

## 5. Compiler (`bestiec`)

### Responsibility

`bestiec` is responsible for **compilation only**.

Examples:

bestiec main.bst
bestiec main.bst -o app
bestiec main.bst --debug
bestiec main.bst --release

Rules:

- No implicit test execution
- No formatting
- No dependency resolution
- No automation logic

---

## 6. Standard Tools

### 6.1 `bestie test`

**Purpose:**
Execute tests defined in the project.

Behavior:

- Discovers tests under `/test`
- Uses `bestie.framework.test`
- Fails on first error by default

Examples:

bestie test
bestie test --watch
bestie test --filter UserService

Configuration is defined in `bestie-project.toml`.

---

### 6.2 `bestie fmt`

**Purpose:**
Format Bestie source code.

Principles:

- Single canonical format
- No user-defined style rules
- Formatting equals spec compliance

Examples:

bestie fmt
bestie fmt ./src

---

### 6.3 `bestie dbg`

**Purpose:**
Interactive debugging of Bestie programs.

Capabilities (initial scope):

- Breakpoints
- Step execution
- Stack inspection
- Variable inspection

Example:

bestie dbg ./app

---

### 6.4 `bestie build`, `bestie run`, `bestie task`

**Purpose:**
Build the project's targets, run them, and run declared automation.

Examples:

bestie build
bestie build --release
bestie run -- --port 8080
bestie task gen-schema
bestie clean

Configuration lives in `bestie-project.toml` — `[[target]]`, `[profile.*]`, and `[tasks]`. There is no separate build file. Tasks run only when invoked, never as a side effect of a build.

Full specification: `package-manager.md` §3, §4.6, §4.7, §6.

---

### 6.5 `bestie doc`

**Purpose:**
Generate documentation from Bestie source code.

Outputs:

- HTML
- Markdown

Sources:

- Public APIs
- Comments
- Type definitions

Example:

bestie doc

---

### 6.6 Project and Dependency Management

**Purpose:**
Create projects, manage dependencies, and publish.

Responsibilities:

- Create projects (`bestie new`, `bestie init`)
- Add, remove, and update dependencies with minimal version selection
- Maintain and verify `bestie.lock`
- Publish to and yank from a registry

Rules:

- `std-lib` and `std-api` ship with the toolchain and are never fetched; the manifest's `bestie` version selects them
- `std-framework` projects are fetched from the registry under the reserved `bestie/` namespace
- No code from a dependency runs during fetch or build

Examples:

bestie new orders --lib
bestie add acme/money@2.1
bestie update
bestie tree
bestie publish

Full specification: `package-manager.md`.

---

### 6.7 `bestie lint` (Reserved)

**Purpose:**
Static analysis beyond formatting.

Potential checks:

- Unused code
- Suspicious constructs
- Best-practice violations

This tool is reserved for future versions.

---

## 7. Explicit Non-Goals

`std-tools` intentionally excludes:

- REPL environments
- Code generators
- Framework-specific CLIs
- IDE integrations
- Runtime reflection tooling

---

## 8. Design Philosophy

1. Explicit over implicit
2. Few tools, clearly scoped
3. Compiler is not a task runner
4. Tools do not shape the language
5. No accidental complexity

---

## 9. Stability Expectations

| Component | Stability |
|---------|----------|
| core | Sealed |
| std-lib | Very stable |
| std-api | Evolves |
| std-framework | Replaceable |
| std-tools | Evolves pragmatically |

---

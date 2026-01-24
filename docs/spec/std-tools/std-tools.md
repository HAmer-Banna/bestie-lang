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

yaml
Copy code

---

## 5. Compiler (`bestiec`)

### Responsibility

`bestiec` is responsible for **compilation only**.

Examples:

bestiec main.bst
bestiec main.bst -o app
bestiec main.bst --debug
bestiec main.bst --release

yaml
Copy code

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
- Uses `std-framework.test`
- Fails on first error by default

Examples:

bestie test
bestie test --watch
bestie test --filter UserService

yaml
Copy code

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

yaml
Copy code

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

yaml
Copy code

---

### 6.4 `bestie build`

**Purpose:**  
Project automation and task orchestration.

Intent:

- Bestie-native alternative to Make
- Cross-platform by default
- Understands Bestie project structure

Examples:

bestie build
bestie build release
bestie build clean

makefile
Copy code

Configuration:

bestie.build.toml

yaml
Copy code

Compatibility with `Makefile` is allowed but optional.

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

yaml
Copy code

---

### 6.6 `bestie mod`

**Purpose:**  
Dependency and module management.

Responsibilities:

- Initialize projects
- Fetch third-party libraries
- Maintain lockfiles

Rules:

- `std-lib`, `std-api`, and `std-framework` are never fetched remotely
- Only external libraries are managed

Examples:

bestie mod init
bestie mod add my-lib
bestie mod update

yaml
Copy code

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

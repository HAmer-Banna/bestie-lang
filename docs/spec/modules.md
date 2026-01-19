Bestie Modules & Project Structure

This document defines how code is organized, compiled, exported, and imported in Bestie.
The goal is clarity, fast compilation, strong encapsulation, and zero magic.

⸻

1. Fundamental Principles

Bestie modules are designed around the following rules:
	1.	Modules are the unit of compilation
	2.	Packages are the unit of namespace
	3.	Exports are explicit
	4.	Imports are explicit
	5.	No circular dependencies between modules
	6.	No implicit re-export
	7.	No classpath scanning, reflection, or runtime discovery

Everything is resolved at compile time.

⸻

2. Project Structure

A Bestie project has a single root and one or more modules.

my-project/
├── bestie.toml
├── modules/
│   ├── core/
│   │   ├── module.toml
│   │   └── src/
│   │       └── core/
│   │           └── math.bst
│   ├── net/
│   │   ├── module.toml
│   │   └── src/
│   │       └── net/
│   │           └── http.bst
│   └── app/
│       ├── module.toml
│       └── src/
│           └── app/
│               └── main.bst


⸻

3. Module

A module is:
	•	A compilation boundary
	•	A dependency boundary
	•	A visibility boundary

3.1 Module Definition

Each module defines itself using module.toml:

name = "net"
version = "1.0.0"

depends = [
  "core"
]

Rules:
	•	Module names are globally unique within the project
	•	Dependencies must form a DAG
	•	Transitive dependencies are allowed but not re-exported

⸻

4. Package

A package is a namespace inside a module.

module: net
package: net.http.client

Declared at the top of a file:

package net.http.client

Rules:
	•	Packages do not imply visibility
	•	Packages never cross module boundaries
	•	Folder structure must match package structure

⸻

5. Visibility Modifiers

Bestie supports four visibility levels:

Modifier	Scope
pub	Visible outside the module
pkg	Visible inside the package
protec	Visible to subclasses
priv	Visible inside the declaring type

Key Rule

Only pub symbols may be exported from a module

⸻

6. Export Rules

6.1 Explicit Export

A symbol is exported only if:
	•	It is declared pub
	•	It belongs to a module

Example:

pub fun parseUrl(input: str): Url { ... }

pub data class Url(
  val scheme: str,
  val host: str
)

6.2 What Cannot Be Exported

The following cannot be exported:
	•	priv members
	•	pkg members
	•	Local classes
	•	Local functions
	•	Lambdas
	•	Inner classes unless explicitly pub

⸻

7. Import Rules

7.1 Importing Modules

Modules are imported implicitly via dependency declaration, not via syntax.

If module.toml declares:

depends = ["net"]

Then code may import exported symbols from net.

⸻

7.2 Import Syntax

import net.http.client.HttpClient
import net.http.client.*

Rules:
	•	Imports are file-scoped
	•	Wildcard imports import only pub symbols
	•	No aliasing (as) for packages (only types/functions)

⸻

7.3 Explicitness Rule

Bestie does not allow:
	•	Implicit imports
	•	Default imports (except core primitives)
	•	Star imports across modules without dependency

⸻

8. Default Imports

The following are always available:

core.lang.*
core.types.*

This includes:
	•	int, float, str
	•	ptr, option
	•	basic annotations

Everything else must be imported.

⸻

9. Re-exporting (Forbidden)

Bestie does not allow re-exporting dependencies.

If:
	•	app depends on net
	•	net depends on core

Then:
	•	app cannot access core unless it declares it

This prevents:
	•	Dependency leakage
	•	Hidden coupling
	•	Unstable APIs

⸻

10. Cycles (Forbidden)

Module dependency cycles are compile-time errors.

A -> B -> C -> A ❌

Reason:
	•	Breaks compilation order
	•	Breaks memory layout guarantees
	•	Breaks optimization assumptions

⸻

11. Local & Internal Code

11.1 Local Classes

Classes declared inside functions:

fun f() {
  class Temp { }
}

Rules:
	•	Always priv
	•	Never exported
	•	No annotations affecting visibility
	•	Cannot implement pub protocols

⸻

12. Std-lib vs User Modules

Aspect	Core	Std-lib	User
Module system	Same	Same	Same
Export rules	Strict	Strict	Strict
Reflection	❌	❌	❌
Dynamic loading	❌	❌	❌

Bestie has one model, not two.

⸻

13. Design Rationale

This model ensures:
	•	Predictable builds
	•	Fast compilation
	•	Stable APIs
	•	No classpath hell
	•	No runtime surprises
	•	Perfect fit with native compilation

⸻

14. Summary
	•	Modules compile, packages organize
	•	Exports are explicit
	•	Imports are explicit
	•	No re-exports
	•	No cycles
	•	No magic

This structure supports Bestie’s core promise:

System-level power with backend-level ergonomics — without compromise.

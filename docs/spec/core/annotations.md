# Annotations

Annotations in Bestie are **compile-time only constructs**. They exist solely to guide the compiler, tooling, and static analysis, and **introduce zero runtime cost**. No annotation metadata is retained in the generated binary unless explicitly materialized by compiler plugins.

---

## Compile-Time Semantics

* Annotations are evaluated and consumed entirely at **compile time**.
* They do **not** generate runtime reflection data.
* They do **not** incur memory overhead or execution penalties.
* Their primary roles include:

  * Validation
  * Code generation
  * Static guarantees
  * Optimization hints

This design aligns annotations with Bestie’s philosophy of *compile-time determinism*.

---

## User-Defined Annotations

Bestie allows **user-defined annotations**, provided they remain strictly compile-time.

Although the Bestie compiler itself is **closed-source and not modifiable**, the language supports **annotation extensions via compiler plugins**. These plugins may:

* Interpret custom annotations
* Enforce additional compile-time rules
* Generate code or metadata
* Integrate with external tooling

This enables rich ecosystems (e.g. frameworks) without compromising compiler stability or runtime performance.

---

## Annotation Syntax

### Definition

```bestie
annotation ValidateRange(min: int, max: int);
```

### Usage

```bestie
@ValidateRange(min = 0, max = 100)
fun setScore(score: int): void;
```

Notes:

* Annotation parameters are **named and typed**.
* All values must be **compile-time constants**.
* Annotations may be applied multiple times unless restricted by the annotation definition.

---

## Predefined Annotations

Bestie ships with a set of **built-in annotations** understood by the compiler and standard tooling:

* `@immutable` — Marks a type or value as deeply immutable
* `@virtual` — Allows dynamic dispatch override
* `@override` — Enforces method override correctness
* `@inline` — Requests function inlining
* `@pure` — Declares a function as side-effect free
* `@partial` — Marks incomplete or intentionally partial definitions
* `@expose` — Exposes an element for external tooling or APIs
* `@noNew` — Prevents direct allocation via `new`
* `@noInit` — Disallows initialization blocks
* `@noConstruct` — Prevents construction entirely

The exact semantics of each annotation are enforced at compile time.

---

## Annotation Targets

Annotations may be applied to:

* Classes
* Protocols
* Functions
* Other annotations

### Annotation-on-Annotation (Composition)

When an annotation is applied to another annotation, it behaves as **annotation composition**:

* The annotated annotation implicitly carries **both its own behavior and its parent’s behavior**.
* This models a form of *inheritance-like reuse* without introducing runtime hierarchies.

Example:

```bestie
@immutable
annotation ValueObject;
```

Any usage of `@ValueObject` now also implies `@immutable`.

---

## Usage in the Standard Framework

Annotations are **extensively used** across Bestie’s standard framework, including but not limited to:

* Web frameworks
* ORM and persistence layers
* Dependency Injection systems
* Validation and schema generation

Because annotations are compile-time only, these frameworks achieve:

* Zero reflection overhead
* Strong static guarantees
* Predictable performance characteristics

---

## Design Rationale

Bestie annotations are designed to be:

* Static rather than reflective
* Explicit rather than magical
* Extensible without compiler modification

This ensures annotations remain a **language-level tool**, not a runtime liability.

# Bestie Low-Level Pillars (Locked)

These pillars are **non-negotiable**.
Any change that violates them is considered a **design regression**, regardless of feature demand.

---

## Pillar 1 — Compilation Speed Is a First-Class Requirement

From day one and forever, **compilation speed is critical**.

### Meaning
- Fast builds are not an optimization target; they are a **hard constraint**
- The compiler must remain usable in:
  - Large monorepos
  - CI pipelines
  - Iterative development loops
- Compile times should scale **linearly with code size** whenever possible

### Implications
- No global type inference requiring whole-program analysis
- No complex lifetime solvers (Rust-style)
- No template explosion (C++)
- No macro systems that expand combinatorially
- No hidden multi-phase resolution

### Policy
If a language feature significantly degrades compilation speed, it is **rejected** — even if expressive.

---

## Pillar 2 — Compiler Engineering Is Part of the Language Contract

Compiler optimization is **not optional** — it is part of Bestie’s design.

### Meaning
- Bestie is designed *with the compiler*, not layered on top of it
- Language semantics are chosen to be:
  - Predictable
  - Analyzable
  - Optimizable
- The compiler is expected to perform **aggressive but safe transformations**

### Implications
- Explicit ownership enables optimization
- No hidden allocation enables escape and lifetime analysis
- No null improves control-flow and layout guarantees
- Sealed core preserves optimization stability

### Policy
If a feature cannot be reliably optimized, it does **not belong in the core**.

---

## Pillar 3 — Memory Layout Is a Strategic Advantage

Bestie guarantees **predictable and optimized memory layout**.

### Meaning
Bestie prioritizes:

- Cache locality
- Minimal metadata and headers
- Efficient alignment and packing
- Reduced pointer indirection

The programmer expresses structure;
the compiler determines layout within semantic guarantees.

### Positioning

- C provides control but limited guarantees
- Rust provides safety with constrained layout freedom
- Zig provides explicitness but requires manual decisions
- Managed languages hide layout behind a runtime

**Bestie combines:**

> Semantic knowledge + compiler-driven layout optimization

### Policy

The compiler may reorder, inline, flatten, and compact memory **when semantic guarantees are preserved**.

Layout optimization is:
- Always enabled
- Transparent to the programmer
- Forward-compatible

---

## Pillar 4 — Machine Code Quality Matters

Bestie treats code generation as a **core engineering responsibility**.

### Meaning
Bestie aims to produce consistently high-quality native code.

### Implications
- Continuous backend improvement
- Profile-guided optimization (PGO)
- Architecture-aware lowering
- Instruction scheduling awareness
- Cache-friendly code generation

### Policy
Bestie continuously improves its machine code quality while preserving semantic stability and predictability.

---

## Core Invariants

These pillars imply permanent language guarantees:

- No hidden allocation in core semantics
- Static dispatch by default
- No runtime reflection
- Ownership validated at compile time
- Predictable object layout rules
- Explicit concurrency model

These invariants remain stable across language versions and form the foundation of Bestie’s design.

---

## What This Means for Developers

- Build times remain predictable and scalable
- Performance characteristics are visible in code
- Memory behavior is deterministic
- Optimizations do not change program semantics
- Programs remain stable and predictable over time

---

## What These Pillars Explicitly Reject

Bestie will not compromise these pillars for:

- Novel syntax
- Paradigm purity
- Academic experimentation
- Trend-driven features
- Hidden runtime abstractions

---

## Summary (Canonical)

Bestie is designed to:

- Compile fast
- Enable aggressive optimization
- Produce predictable memory layout
- Generate high-quality machine code

**Everything else is secondary.**

---

## Strategic Consequence

These pillars imply:

- Bestie is not a scripting-first language
- Bestie is not a research language
- Bestie is not macro-driven
- Bestie is not runtime-centered

Bestie is a **deterministic, compiler-driven native language** that also supports **high-performance backend systems**.

# `bestie.framework.aop`

Aspect-oriented utilities with compile-time weaving only.

## Purpose

`aop` provides cross-cutting behavior composition (logging, metrics, policy checks, tracing) without runtime interception or reflection proxies.

It exists to:

- Keep cross-cutting concerns modular
- Preserve predictable call paths
- Avoid runtime overhead from dynamic weaving

## Layering and Dependencies

`aop` is part of `std-framework` and integrates with compiler-supported transformation hooks and annotations.

Import style:

```bestie
import bestie.framework.aop
```

## Core Concepts

- `Aspect`: reusable concern definition.
- `Pointcut`: compile-time match rules for join points.
- `Advice`: code injected before/after/around matched targets.
- `Join Point`: supported target location (function, method, constructor boundary, etc.).

## Weaving Model

- Weaving occurs at compile time only.
- Generated code is inspectable in build artifacts.
- Failures in pointcut resolution are compile-time errors.
- Aspect ordering is explicit and deterministic.

## Supported Use Cases

- Structured logging around service boundaries
- Metrics/timing instrumentation
- Security policy guards
- Transaction boundary helpers

## Non-Goals

- No runtime proxy generation
- No dynamic weaving after compilation
- No hidden mutation of function signatures

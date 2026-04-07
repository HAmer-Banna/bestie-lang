# `bestie.framework.di`

Dependency injection and inversion-of-control framework with explicit wiring.

## Purpose

`di` manages object construction and dependency graphs while keeping bindings explicit and compile-verifiable where possible.

It is intended to:

- Reduce manual wiring boilerplate
- Centralize lifecycle management
- Improve testability through replaceable bindings

## Layering and Dependencies

`di` is a `std-framework` module that can be used by other frameworks (`web`, `orm`, `gui`, `stream`) and application code.

Import style:

```bestie
import bestie.framework.di
```

## Core Concepts

- `Container`: registry and resolver for bindings.
- `Binding`: mapping from abstraction to implementation/factory.
- `Scope`: lifecycle model (singleton, scoped, transient).
- `Module`: grouped binding declarations.
- `Provider`: function/object that creates instances.

## Resolution Rules

- Constructor injection is preferred.
- Ambiguous bindings are compile/startup errors.
- Circular dependencies are rejected unless explicitly broken by provider indirection.
- Scope violations (for example singleton -> request-scoped direct dependency) are validated.

## Minimal Example

```bestie
import bestie.framework.di

fun buildContainer() -> di::Container {
    let c = di::container()
    c.bind(Logger).to(ConsoleLogger).singleton()
    c.bind(UserService).to_factory(fun(r) { return UserService(r.get(Logger)) })
    return c
}
```

## Non-Goals

- No reflection-based auto-wiring by default
- No hidden global service locator usage
- No runtime mutation of frozen production bindings

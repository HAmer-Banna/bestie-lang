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
- `Scope`: lifecycle model (`@Singleton`, `@Scoped`, `@Transient`).
- `Module`: grouped binding declarations.
- `Provider`: function/object that creates instances.

## Annotation-Driven Wiring

Annotations allow types to declare their own wiring requirements. The compiler validates the dependency graph at compile time — missing bindings, ambiguous types, and scope violations are errors before the binary is produced.

### Component Annotations

| Annotation | Target | Effect |
|---|---|---|
| `@Injectable` | class | Marks as injectable; constructor params are auto-resolved |
| `@Singleton` | class | One instance per container lifetime |
| `@Scoped` | class | One instance per named scope (e.g. per request) |
| `@Transient` | class | New instance on every resolution |
| `@Inject` | constructor or field | Explicit injection point (required when multiple constructors exist) |
| `@Named(name)` | class or param | Disambiguates multiple bindings of the same type |

### Example

```bestie
import bestie.framework.di

@Injectable
@Singleton
class ConsoleLogger : Logger {
    fun log(msg: str) { io::println(msg) }
}

@Injectable
@Scoped
class UserService(@Named("primary") val db: Database, val logger: Logger) {
    fun findById(id: int) -> User? { ... }
}
```

The container resolves `UserService` by injecting the `@Singleton` `ConsoleLogger` and the `@Named("primary")` database binding automatically.

## Module-Based Manual Wiring

For cases where annotation-driven wiring is insufficient (third-party types, complex factories):

```bestie
import bestie.framework.di

fun buildContainer() -> di::Container {
    let c = di::container()

    c.bind(Logger).to(ConsoleLogger).singleton()
    c.bind(Database).named("primary").to_factory(fun(r) {
        return PostgresDatabase(url: env("DB_URL"))
    }).singleton()
    c.bind(UserService).to(UserService).scoped("request")

    return c
}
```

Both styles can be mixed: annotated types are auto-registered when the container scans a module, while manual bindings override or supplement them.

## Resolution Rules

- Constructor injection is preferred.
- `@Inject` is required when a class has multiple constructors.
- Ambiguous bindings (same type, no `@Named` to disambiguate) are compile/startup errors.
- Circular dependencies are rejected unless explicitly broken by provider indirection.
- Scope violations (e.g. `@Singleton` depending directly on `@Scoped`) are validated.

## Non-Goals

- No reflection-based auto-wiring by default
- No hidden global service locator usage
- No runtime mutation of frozen production bindings

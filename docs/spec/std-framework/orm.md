# `bestie.framework.orm`

Database abstraction framework for typed persistence and query workflows.

## Purpose

`orm` provides structured mapping between domain models and relational data while preserving explicit query intent and transaction boundaries.

It aims to reduce boilerplate around:

- Entity persistence
- Query construction
- Transaction management
- Repository organization

## Layering and Dependencies

`orm` is part of `std-framework` and builds on:

- `bestie.api.db`
- `bestie.api.io` (optional for migration scripts)
- `bestie.lib.utilities` / `bestie.lib.collections`

Import style:

```bestie
import bestie.framework.orm
```

## Core Concepts

- `Entity`: persistence-aware domain type.
- `Repository`: typed data access interface.
- `Session`: unit of interaction with the DB backend.
- `Transaction`: explicit boundary for consistent writes.
- `Query`: composable and typed query object.

## Mapping Model

- Field-to-column mappings are declared explicitly.
- Primary keys and relationships are defined in schema metadata.
- Lazy loading is opt-in and explicit.
- Batch operations are supported without implicit hidden flushes.

## Transaction Model

- Transactions are explicit (`begin`, `commit`, `rollback`).
- Nested transactions use savepoints when supported by the driver.
- Auto-commit behavior must be opt-in.

## Minimal Example

```bestie
import bestie.framework.orm

fun createUser(repo: UserRepository, email: str) {
    orm::transaction(fun(tx) {
        repo.insert(User { email: email }, tx)
    })
}
```

## Non-Goals

- No runtime schema guessing via reflection
- No hidden write flush at arbitrary points
- No bypass of explicit transaction semantics

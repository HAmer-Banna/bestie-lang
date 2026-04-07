# `bestie.framework.test`

Testing framework for unit, integration, and behavior-level verification.

## Purpose

`test` provides a consistent structure for writing deterministic tests and integrating with `bestie test` tooling.

Primary goals:

- Fast unit tests
- Isolated integration tests
- Readable assertions and fixtures
- Predictable parallel execution

## Layering and Dependencies

`test` is a `std-framework` package that depends mainly on:

- `core`
- `std-lib.utilities`
- optional `std-api` modules based on test scope

Import style:

```bestie
import bestie.framework.test
```

## Core Concepts

- `Suite`: logical group of related tests.
- `Case`: single named test function.
- `Fixture`: setup/teardown container.
- `Assert`: expectation utilities with rich failure output.
- `Mock`/`Stub`: explicit doubles for integration boundaries.

## Execution Model

- Tests are discoverable by explicit registration, not reflection scanning.
- Isolation mode is configurable (process-level or module-level).
- Parallel runs are enabled where tests declare thread-safety.
- Flaky retries are opt-in and separately reported.

## Minimal Example

```bestie
import bestie.framework.test

test::case("sum adds two integers", fun() {
    let result = sum(2, 3)
    test::assert_eq(result, 5)
})
```

## Non-Goals

- No hidden global ordering assumptions
- No implicit network/filesystem access in unit tests
- No magic auto-mocking by runtime instrumentation

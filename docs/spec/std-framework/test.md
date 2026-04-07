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
- `bestie.lib.util`
- optional `bestie.api` modules based on test scope

Import style (explicit per-symbol):

```bestie
import bestie.framework.test.Suite
import bestie.framework.test.Test
import bestie.framework.test.BeforeEach
import bestie.framework.test.assert_eq
import bestie.framework.test.assert_true
```

## Core Concepts

- `Suite`: logical group of related tests, declared with `@Suite`.
- `Case`: single named test function, declared with `@Test`.
- `Fixture`: setup/teardown container (`@BeforeEach`, `@AfterEach`, `@BeforeAll`, `@AfterAll`).
- `Assert`: expectation functions with rich failure output.
- Fakes/stubs: explicit hand-written doubles for integration boundaries.

## Annotation Reference

| Annotation | Target | Effect |
|---|---|---|
| `@Suite` | class | Declares a test suite; methods annotated with `@Test` are auto-registered |
| `@Test` | method | Registers a test case; display name defaults to method name |
| `@Test(name)` | method | Registers with an explicit display name |
| `@BeforeEach` | method | Runs before every test in the suite |
| `@AfterEach` | method | Runs after every test in the suite |
| `@BeforeAll` | method | Runs once before the suite begins |
| `@AfterAll` | method | Runs once after all tests complete |
| `@Skip(reason)` | method | Skips the test; reason is shown in the report |
| `@Parallel` | class or method | Permits concurrent execution |
| `@Timeout(ms)` | method | Fails the test if it exceeds the given duration |

The compiler generates the test registry from these annotations — no reflection scanning occurs at runtime.

## Example

```bestie
import bestie.framework.test.Suite
import bestie.framework.test.Test
import bestie.framework.test.BeforeEach
import bestie.framework.test.Skip
import bestie.framework.test.assert_eq
import bestie.framework.test.assert_true
import bestie.framework.test.assert_throws

@Suite
class UserServiceTests {

    var service: UserService
    var repo: FakeUserRepository

    @BeforeEach
    fun setup(): void {
        repo = FakeUserRepository.new()
        service = UserService.new(repo)
    }

    @Test("creates a user with the given email")
    fun createUser(): void {
        val user = service.create("ada@example.com")
        assert_eq(user.email, "ada@example.com")
        assert_true(repo.contains(user.id))
    }

    @Test
    fun rejectsBlankEmail(): void {
        assert_throws(fun(): void {
            service.create("")
        })
    }

    @Test
    @Skip("pending payment integration")
    fun chargesUserOnSignup(): void { ... }
}
```

## Assertion Functions

```bestie
assert_eq(actual, expected)
assert_ne(actual, expected)
assert_true(expr)
assert_false(expr)
assert_throws(fun(): void { ... })
```

Failures include the expression source, expected and actual values, and the call site.

## Explicit Test Doubles

Fakes are hand-written explicit implementations — no magic auto-mocking:

```bestie
class FakeUserRepository impl UserRepository {

    var inserted: list<User> = list<User>.build()

    fun findById(id: int): User ? {
        for (u in inserted) {
            if (u.id == id) { return u }
        }
        return
    }

    fun insert(user: User, tx: Transaction): void {
        inserted.add(user)
    }

    fun contains(id: int): bool {
        for (u in inserted) {
            if (u.id == id) { return true }
        }
        return false
    }
}
```

## Execution Model

- Tests are discovered via `@Suite` / `@Test`; the compiler generates the registry (no reflection).
- Isolation mode is configurable (process-level or module-level).
- Parallel runs are enabled where tests or suites declare `@Parallel`.
- Flaky retries are opt-in and separately reported.

## Non-Goals

- No hidden global ordering assumptions
- No implicit network/filesystem access in unit tests
- No magic auto-mocking by runtime instrumentation

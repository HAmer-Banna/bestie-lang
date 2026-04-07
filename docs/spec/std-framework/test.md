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

- `Suite`: logical group of related tests, declared with `@Suite`.
- `Case`: single named test function, declared with `@Test`.
- `Fixture`: setup/teardown container (`@BeforeEach`, `@AfterEach`, `@BeforeAll`, `@AfterAll`).
- `Assert`: expectation utilities with rich failure output.
- `Mock`/`Stub`: explicit doubles for integration boundaries.

## Annotation Reference

| Annotation | Target | Effect |
|---|---|---|
| `@Suite` | class | Declares a test suite; methods annotated with `@Test` are auto-registered |
| `@Test` | method | Registers a test case; name defaults to method name |
| `@Test(name)` | method | Registers with an explicit display name |
| `@BeforeEach` | method | Runs before every test in the suite |
| `@AfterEach` | method | Runs after every test in the suite |
| `@BeforeAll` | method | Runs once before the suite begins |
| `@AfterAll` | method | Runs once after all tests complete |
| `@Skip(reason)` | method | Skips the test; reason is shown in the report |
| `@Parallel` | class or method | Permits concurrent execution |
| `@Timeout(ms)` | method | Fails the test if it exceeds the given duration |

## Example

```bestie
import bestie.framework.test

@Suite
class UserServiceTests {

    val service: UserService
    val repo: FakeUserRepository

    @BeforeEach
    fun setup() {
        repo = FakeUserRepository()
        service = UserService(repo)
    }

    @Test("creates a user with the given email")
    fun createUser() {
        let user = service.create("ada@example.com")
        test::assert_eq(user.email, "ada@example.com")
        test::assert_true(repo.contains(user.id))
    }

    @Test
    fun rejectsBlankEmail() {
        test::assert_throws(fun() {
            service.create("")
        })
    }

    @Test
    @Skip("pending payment integration")
    fun chargesUserOnSignup() { ... }
}
```

## Assertions

```bestie
test::assert_eq(actual, expected)
test::assert_ne(actual, expected)
test::assert_true(expr)
test::assert_false(expr)
test::assert_null(value)
test::assert_not_null(value)
test::assert_throws(fun() { ... })
test::assert_throws_type<ErrorType>(fun() { ... })
```

Failures include the expression source, expected and actual values, and the call site.

## Mocks and Stubs

Explicit doubles only. No magic auto-mocking.

```bestie
import bestie.framework.test

class FakeUserRepository : UserRepository {
    val store = mutable Map<int, User>()

    fun findById(id: int) -> User? { return store.get(id) }
    fun insert(user: User, tx: orm::Transaction) { store.put(user.id, user) }
    fun contains(id: int) -> bool { return store.has(id) }
}
```

## Execution Model

- Tests are discoverable via `@Suite` / `@Test` without reflection scanning (the compiler generates the registry).
- Isolation mode is configurable (process-level or module-level).
- Parallel runs are enabled where tests or suites declare `@Parallel`.
- Flaky retries are opt-in and separately reported.

## Non-Goals

- No hidden global ordering assumptions
- No implicit network/filesystem access in unit tests
- No magic auto-mocking by runtime instrumentation

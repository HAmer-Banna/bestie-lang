# `bestie.lib.test` — Testing

Unit and integration testing for every Bestie project, run by `bestie test`.

```bestie
import bestie.lib.test
```

## Purpose and Placement

Testing lives in **std-lib**, not std-framework. Every package needs tests from day one — std-lib and std-api included — and `bestie test` is part of the toolchain (`std-tools/package-manager.md`). A test library downloaded separately could not test the libraries it depends on. `bestie.lib.test` uses no OS facility, so it also runs in freestanding builds on a target or in an emulator.

It is built with the same public mechanism any framework uses: an aggregate processor (`core/metaprogramming.md` §3.2). There is no runtime discovery and no reflection.

## 1. Writing Tests

A test is a function with `@Test`. It passes if it returns, and fails if an assertion fails or it returns an error.

```bestie
import bestie.lib.test

@Test
fun parsesPort() {
    expectEq("8080".toInt() catch 0, 8080)
}

@Test("rejects a blank email")
fun rejectsBlankEmail(): void ! TestFailure {
    val own repo    = FakeUserRepository.new()
    val own service = UserService.new(repo: repo)

    expectErr(service.create(""), UserError.InvalidEmail)
}
```

* Tests are **top-level functions**. Shared setup is an ordinary function the test calls, so each test shows everything it depends on. There is no `@BeforeEach` and no suite class whose fields carry hidden state between tests.
* A test may return `void ! E` for any error set. Returning an error fails the test and reports it, so `try` works inside tests.
* Test files are any files in the project's `tests/` directory, or functions marked `@Test` in `src/` under `when (build.test)`, which keeps them out of release builds.

## 2. Annotations

| Annotation | Target | Effect |
| ---------- | ------ | ------ |
| `@Test` / `@Test(name: str)` | function | Registers a test. The name defaults to the function name |
| `@Skip(reason: str)` | function | Registered but not run; the reason is reported |
| `@Timeout(ms: int)` | function | Fails the test if it runs longer |
| `@Serial` | function | Never run concurrently with other tests (for tests that share an external resource) |

The processor checks at compile time that a `@Test` function takes no parameters and returns `void` or `void ! E`.

## 3. Assertions

| Function | Fails when |
| -------- | ---------- |
| `expect(cond: bool)` | `cond` is false |
| `expectEq(actual, expected)` | Not equal. Requires `Equable`; the report shows both values |
| `expectNe(actual, expected)` | Equal |
| `expectAbsent(x: T ?)` | `x` is present |
| `expectPresent(x: T ?): T` | `x` is absent. Returns the value |
| `expectOk(x: T ! E): T` | `x` is an error. Returns the value |
| `expectErr(x: T ! E, err: E)` | `x` succeeded, or failed with a different error |
| `expectPanics(f)` | Calling `f` does not panic. Runs `f` in a child process, since a panic ends a process (`core/exceptions.md`) |
| `fail(msg: str)` | Always |

A failed assertion records the source expression, the values, and the location, then ends the test. Bestie has no exceptions, so there is no `assertThrows`; errors are values and are checked as values.

## 4. Test Doubles

Doubles are hand-written implementations of a protocol. There is no auto-mocking, which would need runtime instrumentation:

```bestie
class FakeUserRepository impl UserRepository {
    var own users: List<User> = List<User>.new()

    fun find(id: int): User ? {
        for (u in users) {
            if (u.id == id) { return u }
        }
        return
    }

    fun insert(u: User) {
        users.add(u)
    }
}
```

## 5. Running

`bestie test` builds the project with `build.test` set, collects the registry the processor generated for each module, and runs it.

* Tests run in parallel across threads, except `@Serial` ones.
* `bestie test <pattern>` runs the tests whose names match.
* Output is human-readable by default; `--format junit` writes JUnit XML for CI.
* The exit code is non-zero when any test fails.

## 6. Non-Goals

* No suite classes, fixtures with hidden state, or global ordering
* No exception assertions — errors are values
* No runtime mocking or instrumentation
* No implicit network or filesystem access

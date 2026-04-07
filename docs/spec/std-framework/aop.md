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

- `Aspect`: reusable concern definition, declared with `@Aspect`.
- `Pointcut`: compile-time match rule for join points, declared with `@Pointcut`.
- `Advice`: code injected at matched join points (`@Before`, `@After`, `@Around`).
- `Join Point`: supported target location (function, method, constructor boundary).
- `JoinContext`: runtime value passed to advice carrying method metadata and arguments.

## Annotation Reference

| Annotation | Target | Effect |
|---|---|---|
| `@Aspect` | class | Declares this class as an aspect |
| `@Pointcut(expr)` | method | Defines a named, reusable pointcut expression |
| `@Before(pointcut)` | method | Advice runs before each matched join point |
| `@After(pointcut)` | method | Advice runs after each matched join point (always) |
| `@AfterReturn(pointcut)` | method | Advice runs after a successful return |
| `@AfterThrow(pointcut)` | method | Advice runs after an exception propagates |
| `@Around(pointcut)` | method | Advice wraps the join point; must call `proceed()` to continue |

Pointcut expressions are evaluated and resolved at compile time. Unresolved or ambiguous pointcuts are compile-time errors.

## Weaving Model

- Weaving occurs at compile time only.
- Generated code is inspectable in build artifacts.
- Failures in pointcut resolution are compile-time errors.
- Aspect ordering is explicit and deterministic (declared via `@Order(n)`).

## Example: Timing Aspect

```bestie
import bestie.framework.aop

@Aspect
class TimingAspect {

    @Pointcut("@annotation(web.Get) || @annotation(web.Post)")
    fun webHandlers() {}

    @Around("webHandlers()")
    fun measureTime(ctx: aop::JoinContext) -> any {
        let start = DateTime.now()
        let result = ctx.proceed()
        let elapsed = DateTime.now() - start
        metrics::record("handler.latency", elapsed, tags: { route: ctx.methodName })
        return result
    }
}
```

## Example: Logging Aspect

```bestie
import bestie.framework.aop

@Aspect
@Order(10)
class AuditAspect {

    @Pointcut("@annotation(Roles)")
    fun securedMethods() {}

    @Before("securedMethods()")
    fun logAccess(ctx: aop::JoinContext) {
        let principal = ctx.arg<Context>("ctx").principal
        io::println("AUDIT: ${principal.id} called ${ctx.methodName}")
    }

    @AfterThrow("securedMethods()")
    fun logFailure(ctx: aop::JoinContext, err: error) {
        io::println("AUDIT FAIL: ${ctx.methodName} threw ${err}")
    }
}
```

## Supported Use Cases

- Structured logging around service boundaries
- Metrics/timing instrumentation
- Security policy guards
- Transaction boundary helpers

## Non-Goals

- No runtime proxy generation
- No dynamic weaving after compilation
- No hidden mutation of function signatures

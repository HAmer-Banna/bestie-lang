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

Import style (explicit per-symbol):

```bestie
import bestie.framework.aop.Aspect
import bestie.framework.aop.Pointcut
import bestie.framework.aop.Around
import bestie.framework.aop.JoinContext
```

## Core Concepts

- `Aspect`: reusable concern definition, declared with `@Aspect`.
- `Pointcut`: compile-time match rule for join points, declared with `@Pointcut`.
- `Advice`: code injected at matched join points (`@Before`, `@After`, `@Around`).
- `Join Point`: supported target location (function, method, constructor boundary).
- `JoinContext`: value passed to advice carrying the method name and argument list.

## Annotation Reference

| Annotation | Target | Effect |
|---|---|---|
| `@Aspect` | class | Declares this class as an aspect |
| `@Pointcut(expr)` | method | Defines a named, reusable pointcut expression |
| `@Before(pointcut)` | method | Advice runs before each matched join point |
| `@After(pointcut)` | method | Advice runs after each matched join point (always) |
| `@AfterReturn(pointcut)` | method | Advice runs after a successful return |
| `@AfterThrow(pointcut)` | method | Advice runs after an error propagates |
| `@Around(pointcut)` | method | Advice wraps the join point; must call `ctx.proceed()` to continue |
| `@Order(n)` | class | Sets the execution priority when multiple aspects apply (lower = earlier) |

Pointcut expressions are evaluated and resolved at compile time. Unresolved or ambiguous pointcuts are compile-time errors.

## Weaving Model

- Weaving occurs at compile time only.
- Generated code is inspectable in build artifacts.
- Failures in pointcut resolution are compile-time errors.
- Aspect ordering is deterministic via `@Order`.

## Example: Timing Aspect

```bestie
import bestie.framework.aop.Aspect
import bestie.framework.aop.Pointcut
import bestie.framework.aop.Around
import bestie.framework.aop.JoinContext
import bestie.lib.datetime.DateTime

@Aspect
class TimingAspect {

    @Pointcut("@annotation(Get) || @annotation(Post)")
    fun webHandlers(): void {}

    @Around("webHandlers()")
    fun measureTime(ctx: JoinContext): void {
        val start = DateTime.now()
        ctx.proceed()
        val elapsed = DateTime.now() - start
        metrics.record("handler.latency", elapsed, ctx.methodName)
    }
}
```

## Example: Audit Logging Aspect

```bestie
import bestie.framework.aop.Aspect
import bestie.framework.aop.Order
import bestie.framework.aop.Pointcut
import bestie.framework.aop.Before
import bestie.framework.aop.AfterThrow
import bestie.framework.aop.JoinContext

@Aspect
@Order(10)
class AuditAspect {

    @Pointcut("@annotation(Roles)")
    fun securedMethods(): void {}

    @Before("securedMethods()")
    fun logAccess(ctx: JoinContext): void {
        val principal = ctx.arg(Context).principal
        print("AUDIT: ${principal.id} called ${ctx.methodName}")
    }

    @AfterThrow("securedMethods()")
    fun logFailure(ctx: JoinContext): void {
        print("AUDIT FAIL: ${ctx.methodName} raised an error")
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

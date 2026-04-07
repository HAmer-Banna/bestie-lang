# `bestie.framework.web`

HTTP web framework with annotation-driven routing and a servlet-style execution model.

## Purpose

`web` provides structured request/response handling on top of `bestie.api.http`. Its primary value over the raw HTTP API is **annotation-driven routing and controller dispatch**: route registration, path/query/body parameter binding, middleware composition, and response serialization are all handled by compile-time code generation — no boilerplate, no runtime reflection.

It is designed for:

- REST APIs and server-rendered web applications
- Predictable, auditable middleware pipelines
- Controller-style service organization
- Easy integration with `di`, `template`, and `orm`

## Layering and Dependencies

This framework sits in `std-framework` and depends on lower layers only:

- `core`
- `std-lib`
- `bestie.api.http`
- `bestie.api.io` (optional, for static files and streaming bodies)
- `bestie.api.network` (optional, for lower-level transport tuning)

Import style (explicit per-symbol):

```bestie
import bestie.framework.web.App
import bestie.framework.web.Request
import bestie.framework.web.Response
import bestie.framework.web.Context
```

## Core Concepts

- `Controller`: class whose methods are compiled into route handlers.
- `Request` / `Response`: explicit values passed through handlers.
- `Router`: generated at compile time from controller annotations.
- `Middleware`: composable interception step; applied via annotation or explicit registration.
- `Context`: per-request scoped values (trace id, auth principal, locale, etc.).
- `App`: owns listener lifecycle and graceful shutdown hooks.

## Annotation-Driven Routing

Route registration is generated entirely at compile time from annotations on controller classes and their methods. There is no runtime scanning.

### Controller Annotations

| Annotation | Target | Effect |
|---|---|---|
| `@Controller(prefix)` | class | Groups routes under a path prefix |
| `@RestController(prefix)` | class | Like `@Controller` but auto-serializes return values to JSON |
| `@Get(path)` | method | Registers a GET route |
| `@Post(path)` | method | Registers a POST route |
| `@Put(path)` | method | Registers a PUT route |
| `@Delete(path)` | method | Registers a DELETE route |
| `@Patch(path)` | method | Registers a PATCH route |

### Parameter Injection Annotations

| Annotation | Target | Effect |
|---|---|---|
| `@PathParam(name)` | parameter | Binds a path segment (e.g. `/users/:id`) |
| `@QueryParam(name)` | parameter | Binds a query string value |
| `@Body` | parameter | Deserializes the full request body |
| `@Header(name)` | parameter | Binds a specific request header value |
| `@Ctx` | parameter | Injects the current request `Context` |

### Middleware and Security Annotations

| Annotation | Target | Effect |
|---|---|---|
| `@Use(MiddlewareType)` | class or method | Applies a middleware to the controller or a single route |
| `@Auth` | class or method | Requires an authenticated principal in `Context` |
| `@Roles(roles...)` | class or method | Requires one of the listed roles on the principal |

All of these are compile-time — the compiler plugin validates bindings and generates the dispatch wiring. Missing path params, type mismatches, or unknown middleware types are compile-time errors.

## Execution Model

1. Accept connection and parse request (`bestie.api.http`)
2. Build request `Context`
3. Run middleware chain (in declaration order)
4. Dispatch to matched controller method
5. Serialize return value to `Response`
6. Write response to socket
7. Finalize request-scoped resources

All phases are explicit and observable in user code.

## Controller Example

```bestie
import bestie.framework.web.RestController
import bestie.framework.web.Use
import bestie.framework.web.Auth
import bestie.framework.web.Roles
import bestie.framework.web.Get
import bestie.framework.web.Post
import bestie.framework.web.Delete
import bestie.framework.web.PathParam
import bestie.framework.web.QueryParam
import bestie.framework.web.Body
import bestie.framework.web.NoContent

@RestController("/users")
@Use(LoggingMiddleware)
@Auth
class UserController(val service: UserService) {

    @Get("/")
    fun list(@QueryParam("page") page: int): list<UserDto> {
        return service.listUsers(page)
    }

    @Get("/:id")
    fun get(@PathParam("id") id: str): UserDto {
        return service.findById(id)
    }

    @Post("/")
    fun create(@Body req: CreateUserRequest): UserDto {
        return service.createUser(req)
    }

    @Delete("/:id")
    @Roles("admin")
    fun delete(@PathParam("id") id: str): NoContent {
        service.deleteUser(id)
        return NoContent.new()
    }
}
```

The compiler generates all route registrations, parameter extraction, and response serialization from the annotations above.

## Minimal Imperative Example

For simple scripts or cases where annotation-style is unnecessary:

```bestie
import bestie.framework.web.App
import bestie.framework.web.Request
import bestie.framework.web.Response
import bestie.framework.web.Context

fun main(): void {
    val app = App.new()

    app.get("/health", fun(req: Request, ctx: Context): Response {
        return Response.new(200, "ok")
    })

    app.listen("0.0.0.0", 8080)
}
```

## Custom Middleware

```bestie
import bestie.framework.web.Middleware
import bestie.framework.web.Request
import bestie.framework.web.Response
import bestie.framework.web.Context
import bestie.framework.web.Next

class LoggingMiddleware impl Middleware {
    fun handle(req: Request, ctx: Context, next: Next): Response {
        print("→ ${req.method} ${req.path}")
        val res = next.call(req, ctx)
        print("← ${res.status}")
        return res
    }
}
```

## Server Bootstrap with DI

```bestie
import bestie.framework.web.App
import bestie.framework.di.Container

fun main(): void {
    val container = buildContainer()
    val app = App.new()

    app.register(container.get(UserController))
    app.register(container.get(ProductController))

    app.listen("0.0.0.0", 8080)
}
```

## Non-Goals

- No runtime annotation scanning or reflection
- No controller auto-discovery from classpath
- No hidden global mutable state
- No implicit thread-per-request guarantee (strategy is configurable)

## Integration Notes

- Use `di` for constructor injection into controllers (controllers are resolved through the container).
- Use `template` for MVC server-rendered views (return a `View` from a `@Controller` method).
- Use `orm` for repository access inside controller services.
- Use `stream` for chunked or SSE response bodies.

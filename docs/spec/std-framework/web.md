# `bestie.framework.web`

HTTP-based web framework with a servlet-style execution model and minimal core abstractions.

## Purpose

`web` provides structured request/response handling on top of `bestie.api.http` without introducing runtime reflection, hidden dispatch, or VM-style behavior.

It is designed for:

- Predictable HTTP services
- Clear middleware pipelines
- Explicit route-to-handler mapping
- Easy integration with `di`, `template`, and `orm`

## Layering and Dependencies

This framework sits in `std-framework` and depends on lower layers only:

- `core`
- `std-lib`
- `std-api.http`
- `std-api.io` (optional, for static files and streaming bodies)
- `std-api.network` (optional, for lower-level transport tuning)

Import style:

```bestie
import bestie.framework.web
```

## Core Concepts

- `Request` and `Response` are explicit values passed through handlers.
- `Router` maps HTTP method + path to a handler function.
- `Middleware` is a composable interception step around handlers.
- `Context` holds per-request scoped values (trace id, auth principal, locale, etc.).
- `Server` owns listener lifecycle and graceful shutdown hooks.

## Execution Model

The model is intentionally simple and servlet-like:

1. Accept connection and parse request
2. Build request context
3. Run middleware chain
4. Invoke route handler
5. Serialize response and write to socket
6. Finalize request-scoped resources

All phases are explicit and observable in user code.

## Non-Goals

- No runtime annotation scanning
- No controller auto-discovery by reflection
- No hidden global mutable state
- No implicit thread-per-request guarantee (actual strategy is configurable)

## Minimal Example

```bestie
import bestie.framework.web

fun main() {
    let app = web::app()

    app.get("/health", fun(req, ctx) {
        return web::response(200, "ok")
    })

    app.listen("0.0.0.0", 8080)
}
```

## Integration Notes

- Use `di` for service wiring instead of global singletons.
- Use `template` for MVC server-rendered views.
- Use `orm` for repository access inside route services.
- Use `stream` for chunked or reactive response flows.

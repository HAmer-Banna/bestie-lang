# `bestie.framework.template`

MVC template engine framework for server-side rendered views.

## Purpose

`template` provides a deterministic, compile-aware rendering system for HTML/text views used in web applications. It favors explicit model binding and predictable output over dynamic runtime magic.

Primary use cases:

- Server-side HTML rendering
- Shared view layouts and components
- Form rendering and validation feedback
- Email/text template generation

## Layering and Dependencies

`template` belongs to `std-framework` and is typically used with:

- `bestie.framework.web`
- `bestie.lib.format`
- `bestie.api.io` (optional for template loading backends)

Import style:

```bestie
import bestie.framework.template
```

## Core Concepts

- `View`: named render target (for example `users/list`).
- `Model`: explicit key/value data passed to the view.
- `Layout`: outer template wrapping page fragments.
- `Partial`: reusable template section.
- `Renderer`: compiles and renders templates with escape rules.

## Rendering Rules

- Auto-escaping is enabled by default for HTML outputs.
- Raw output requires explicit opt-in markers.
- Missing model keys are compile-time or startup-time errors when possible.
- Rendering is side-effect free by default (except explicit I/O integrations).

## MVC Usage Pattern

Typical flow:

1. Controller/handler gathers domain data
2. Data is mapped to a view model
3. Renderer resolves layout + view + partials
4. Escaped output is returned to `web` response

## Minimal Example

```bestie
import bestie.framework.template

fun renderHome(userName: str) -> str {
    let model = template::model()
    model.put("name", userName)
    return template::render("home/index", model)
}
```

## Non-Goals

- No runtime code generation in production path
- No implicit global model mutation
- No reflection-based field extraction by default

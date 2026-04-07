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

Import style (explicit per-symbol):

```bestie
import bestie.framework.template.ViewModel
import bestie.framework.template.Renderer
```

## Core Concepts

- `View`: named render target (e.g. `"users/list"`).
- `ViewModel`: typed model bound to a view, declared with `@ViewModel`.
- `Layout`: outer template wrapping page fragments.
- `Partial`: reusable template section, declared with `@Partial`.
- `Renderer`: compiles and renders templates with escape rules.

## Annotation Reference

| Annotation | Target | Effect |
|---|---|---|
| `@ViewModel(view)` | class | Binds the class as the expected model type for a named view |
| `@Partial(name)` | class | Marks a model as the bound type for a named partial |
| `@Escapes(mode)` | class | Declares the output escaping mode (`html`, `text`, `none`) |

`@ViewModel` allows the compiler to verify that all fields referenced in the template file exist on the model class. Unknown field references in templates are compile-time errors when the template is pre-compiled.

## Rendering Rules

- Auto-escaping is enabled by default for HTML outputs.
- Raw output requires an explicit `{! raw_expr !}` marker in the template.
- Missing model keys referenced in templates are compile-time errors (with pre-compilation) or startup-time errors.
- Rendering is side-effect free by default (except explicit I/O integrations).

## MVC Usage Pattern

Typical flow:

1. Controller/handler gathers domain data
2. Data is mapped to a `ViewModel`
3. Renderer resolves layout + view + partials
4. Escaped output is returned to `web` response

## Example

```bestie
import bestie.framework.template.ViewModel
import bestie.framework.template.Renderer
import bestie.framework.web.RestController
import bestie.framework.web.Get
import bestie.framework.web.Ctx
import bestie.framework.web.Context
import bestie.framework.web.HtmlResponse

@ViewModel("home/index")
class HomeViewModel {
    val userName: str
    val itemCount: int
}

@RestController("/")
class HomeController(val renderer: Renderer) {

    @Get("/")
    fun home(@Ctx ctx: Context): HtmlResponse {
        val model = HomeViewModel.new(userName: ctx.principal.name, itemCount: 42)
        return HtmlResponse.new(renderer.render(model))
    }
}
```

Template file `home/index.html`:

```html
<h1>Welcome, {{ userName }}</h1>
<p>You have {{ itemCount }} items.</p>
```

## Non-Goals

- No runtime code generation in production path
- No implicit global model mutation
- No reflection-based field extraction

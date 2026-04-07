# `bestie.framework.gui`

Desktop UI framework for native windowed applications.

## Purpose

`gui` provides a structured model for building desktop user interfaces with explicit state updates, event handling, and rendering lifecycles.

It targets:

- Native desktop tools
- Admin/control panels
- Data-entry interfaces
- Internal productivity apps

## Layering and Dependencies

`gui` is in `std-framework` and relies on:

- `bestie.api.os`
- `bestie.api.io`
- `bestie.api.network` (optional for remote data sources)

Import style (explicit per-symbol):

```bestie
import bestie.framework.gui.App
import bestie.framework.gui.View
import bestie.framework.gui.State
import bestie.framework.gui.Node
```

## Core Concepts

- `App`: process-level UI runtime owner.
- `Window`: top-level native container.
- `View`: composable UI component, declared with `@View`.
- `State`: reactive field driving UI re-renders, declared with `@State`.
- `Node`: the result of rendering a view component.
- `Event`: user/system input signal.

## Annotation Reference

| Annotation | Target | Effect |
|---|---|---|
| `@View` | class | Declares a composable view component |
| `@State` | field | Declares a reactive field; mutations trigger a re-render pass |
| `@Bind(expr)` | field or param | Creates a one-way binding from a state expression to a view property |
| `@UIThread` | method | Enforces at compile time that the method runs on the UI thread |

## UI Model

- State mutations are explicit and localized to `@State` fields.
- State changes trigger a re-render pass; the framework diffs and applies only changed nodes.
- Rendering can be immediate or retained mode depending on backend profile.
- Thread-affinity constraints (UI thread access) are enforced by `@UIThread` at compile time.

## Example

```bestie
import bestie.framework.gui.App
import bestie.framework.gui.View
import bestie.framework.gui.State
import bestie.framework.gui.Node
import bestie.framework.gui.column
import bestie.framework.gui.label
import bestie.framework.gui.button

@View
class CounterView {

    @State
    var count: int = 0

    fun render(): Node {
        return column([
            label("Count: ${count}"),
            button("Increment", onClick: fun(): void { count += 1 }),
            button("Reset",     onClick: fun(): void { count = 0 })
        ])
    }
}

fun main(): void {
    val app = App.new("Bestie Desktop")
    val win = app.window("Counter", 400, 300)
    win.root(CounterView.new())
    win.show()
    app.run()
}
```

## Non-Goals

- No hidden global widget registry
- No reflection-driven event wiring
- No implicit cross-thread state mutation

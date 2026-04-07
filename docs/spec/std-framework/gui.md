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

Import style:

```bestie
import bestie.framework.gui
```

## Core Concepts

- `Application`: process-level UI runtime owner.
- `Window`: top-level native container.
- `View`: composable UI node.
- `State`: explicit mutable/immutable model driving UI updates.
- `Event`: user/system input signal.
- `Binding`: reactive link between state and a view property.

## Annotation Reference

| Annotation | Target | Effect |
|---|---|---|
| `@View` | class | Declares a composable view component |
| `@State` | field | Declares a reactive state field; mutations trigger re-render |
| `@Bind(expr)` | field or param | Creates a one-way binding from a state expression to a view property |
| `@OnEvent(EventType)` | method | Registers the method as a handler for the given event type |
| `@UIThread` | method | Asserts (enforced at compile time) that the method runs on the UI thread |

## UI Model

- State mutations are explicit and localized to `@State` fields.
- Event handlers are registered via `@OnEvent` or explicit `view.on(Event, handler)`.
- State changes trigger a re-render pass; the framework diffs and applies only changed nodes.
- Rendering can be immediate or retained mode depending on backend profile.
- Thread-affinity constraints (UI thread access) are enforced by `@UIThread` at compile time.

## Example

```bestie
import bestie.framework.gui

@View
class CounterView {

    @State
    var count: int = 0

    fun render() -> gui::Node {
        return gui::column([
            gui::label("Count: ${count}"),
            gui::button("Increment", @OnEvent(gui::Click) fun() {
                count += 1
            }),
            gui::button("Reset", @OnEvent(gui::Click) fun() {
                count = 0
            })
        ])
    }
}

fun main() {
    let app = gui::app("Bestie Desktop")
    let win = app.window("Counter", 400, 300)
    win.root(CounterView())
    win.show()
    app.run()
}
```

## Non-Goals

- No hidden global widget registry
- No reflection-driven event wiring
- No implicit cross-thread state mutation

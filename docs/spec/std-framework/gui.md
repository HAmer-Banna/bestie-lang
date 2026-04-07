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

## UI Model

- Event handlers are explicit functions.
- State updates are deterministic and testable.
- Rendering can be immediate or retained mode depending on backend profile.
- Thread-affinity constraints (UI thread access) are explicit and enforced.

## Minimal Example

```bestie
import bestie.framework.gui

fun main() {
    let app = gui::app("Bestie Desktop")
    let win = app.window("Main", 960, 640)
    win.show()
    app.run()
}
```

## Non-Goals

- No hidden global widget registry
- No reflection-driven event wiring
- No implicit cross-thread state mutation

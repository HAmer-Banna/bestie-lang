# `bestie.framework.stream`

Streaming/reactive framework for asynchronous data pipelines.

## Purpose

`stream` provides composable primitives for continuous data flow, transformation, and backpressure-aware processing.

Typical use cases:

- Event processing pipelines
- Real-time data transformation
- Message and log processing
- Async fan-in/fan-out workflows
- Server-Sent Events and WebSocket feeds (via `web` integration)

## Layering and Dependencies

`stream` belongs to `std-framework` and builds on:

- `bestie.api.ext.concurrency`
- `bestie.api.io` / `bestie.api.network` (source and sink adapters)
- `bestie.lib.functional`

Import style:

```bestie
import bestie.framework.stream
```

## Core Concepts

- `Publisher<T>`: emits items of type `T`.
- `Subscriber<T>`: consumes items of type `T`.
- `Operator<T, R>`: transforms, filters, or aggregates stream items.
- `Scheduler`: controls execution context for operators.
- `Backpressure`: explicit flow-control contract between publisher and subscriber.

## Annotation Reference

| Annotation | Target | Effect |
|---|---|---|
| `@Source` | method | Declares the method as a stream source factory |
| `@Sink` | method | Declares the method as a stream terminal consumer |
| `@OnError(strategy)` | parameter | Attaches an error handling strategy to a pipeline step |

These are lightweight hints for tooling and documentation; the primary API is the fluent operator chain.

## Processing Model

- Pipelines are built by chaining operators; no items flow until a subscriber attaches.
- Cancellation signals propagate upstream through the entire chain.
- Errors propagate as terminal stream events unless explicitly recovered with `.onError(...)`.
- Time/window operators are deterministic under a configured `Scheduler`.
- Backpressure is enforced: a slow subscriber signals demand, a fast publisher waits or buffers within declared limits.

## Example: Basic Pipeline

```bestie
import bestie.framework.stream

fun run() {
    stream::from([1, 2, 3, 4, 5, 6])
        .filter(fun(x) { return x % 2 == 0 })
        .map(fun(x) { return x * 10 })
        .subscribe(fun(x) { io::println(x) })
}
```

## Example: Fan-out with Merge

```bestie
import bestie.framework.stream

fun mergedEvents(a: Publisher<Event>, b: Publisher<Event>) -> Publisher<Event> {
    return stream::merge(a, b)
        .filter(fun(e) { return e.severity >= Severity.WARN })
        .dedupe(window: Duration.seconds(5))
}
```

## Example: SSE Integration with `web`

```bestie
import bestie.framework.web
import bestie.framework.stream

@RestController("/events")
class EventController(val bus: EventBus) {

    @Get("/live")
    fun liveEvents() -> web::SseResponse {
        return web::sse(bus.subscribe())
    }
}
```

## Non-Goals

- No hidden thread creation policies
- No implicit buffering without declared limits
- No opaque runtime scheduling behavior

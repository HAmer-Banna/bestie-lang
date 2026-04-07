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

Import style (explicit per-symbol):

```bestie
import bestie.framework.stream.Publisher
import bestie.framework.stream.from
import bestie.framework.stream.merge
```

## Core Concepts

- `Publisher<T>`: emits items of type `T`.
- `Subscriber<T>`: consumes items of type `T`.
- `Operator<T, R>`: transforms, filters, or aggregates stream items.
- `Scheduler`: controls execution context for operators.
- `Backpressure`: explicit flow-control contract between publisher and subscriber.

## Processing Model

- Pipelines are built by chaining operators; no items flow until a subscriber attaches.
- Cancellation signals propagate upstream through the entire chain.
- Errors propagate as typed `! E` terminal events unless explicitly recovered with `.onError(...)`.
- Time/window operators are deterministic under a configured `Scheduler`.
- Backpressure is enforced: a slow subscriber signals demand, a fast publisher waits or buffers within declared limits.

## Example: Basic Pipeline

```bestie
import bestie.framework.stream.from

fun run(): void {
    from([1, 2, 3, 4, 5, 6])
        .filter(fun(x: int): bool { return x % 2 == 0 })
        .map(fun(x: int): int { return x * 10 })
        .subscribe(fun(x: int): void { print(x.toStr()) })
}
```

## Example: Fan-out with Merge

```bestie
import bestie.framework.stream.Publisher
import bestie.framework.stream.merge

fun mergedEvents(a: Publisher<Event>, b: Publisher<Event>): Publisher<Event> {
    return merge(a, b)
        .filter(fun(e: Event): bool { return e.severity >= Severity.WARN })
        .dedupe(window: 5)
}
```

## Example: SSE Integration with `web`

```bestie
import bestie.framework.web.RestController
import bestie.framework.web.Get
import bestie.framework.web.SseResponse
import bestie.framework.stream.Publisher

@RestController("/events")
class EventController(val bus: EventBus) {

    @Get("/live")
    fun liveEvents(): SseResponse {
        return SseResponse.new(bus.subscribe())
    }
}
```

## Non-Goals

- No hidden thread creation policies
- No implicit buffering without declared limits
- No opaque runtime scheduling behavior

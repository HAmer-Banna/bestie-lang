# `bestie.framework.stream`

Streaming/reactive framework for asynchronous data pipelines.

## Purpose

`stream` provides composable primitives for continuous data flow, transformation, and backpressure-aware processing.

Typical use cases:

- Event processing pipelines
- Real-time data transformation
- Message and log processing
- Async fan-in/fan-out workflows

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

- `Publisher`: emits items.
- `Subscriber`: consumes items.
- `Operator`: transforms/filter/aggregates stream items.
- `Scheduler`: controls execution context.
- `Backpressure`: explicit flow-control contract.

## Processing Model

- Pipelines are built by chaining operators.
- Cancellation signals propagate upstream.
- Errors propagate as terminal stream events unless explicitly recovered.
- Time/window operators are deterministic under a configured scheduler.

## Minimal Example

```bestie
import bestie.framework.stream

fun run() {
    stream::from([1, 2, 3, 4])
        .filter(fun(x) { return x % 2 == 0 })
        .map(fun(x) { return x * 10 })
        .subscribe(fun(x) { io::println(x) })
}
```

## Non-Goals

- No hidden thread creation policies
- No implicit buffering without declared limits
- No opaque runtime scheduling behavior

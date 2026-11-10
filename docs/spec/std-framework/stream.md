# `bestie.framework.stream`

Asynchronous streams of values with backpressure, in the spirit of Project Reactor and Kotlin Flow, built on fibers and channels.

```toml
[dependencies]
"bestie/stream" = "1.0"
```

```bestie
import bestie.framework.stream
```

## Purpose

Backends process data that arrives over time: messages from a broker, rows from a cursor, events from a socket, ticks from a clock. `stream` gives them one composable type, `Flow<T, E>`, with the operators engineers know from Reactor, RxJava, and Kotlin Flow — `map`, `filter`, `buffer`, `merge`, `zip`, `window`, `debounce`, `retry` — and connects it to `bestie/web` for server-sent events.

It is deliberately smaller than Reactor. Bestie has fibers, so a stream can be **pull-based**: a slow consumer simply makes the producer wait. There is no `request(n)` protocol, no subscription objects, and no scheduler zoo. Backpressure is a blocked call, exactly as on a full `Channel` (`std-lib/concurrency.md` §4).

## Layering

| Depends on | For |
| ---------- | --- |
| `core` | Generics, `T ! E`, ownership |
| `bestie.lib.concurrency` | Fibers, `Channel<T>`, cancellation |
| `bestie.lib.datetime` | `Duration` for time-based operators |

`bestie/web` depends on `stream` only for `SseStream.from(flow)`; neither requires the other.

## 1. The Model

A `Flow<T, E>` is a **cold** recipe: nothing runs until a terminal operation collects it, and each collection runs it again from the start. It emits zero or more values of type `T`, then completes, or fails with an error `E`. A flow that cannot fail has `E = never`.

```bestie
fun numbers(): Flow<int, never> {
    return flow(emit => {
        for (i in 0..10) {
            try emit(i)                      // waits while downstream is busy
        }
    })
}

fun main() {
    numbers()
        .filter(n => n % 2 == 0)
        .map(n => n * n)
        .collect(n => println(n.toStr()))
}
```

* **`emit(v)` returns `void ! Stopped`.** `Stopped` means the collector has finished early (`take`, `first`, cancellation). `try emit(v)` ends the producer cleanly; `Stopped` never reaches the collector.
* **Ownership flows downstream.** An `own T` element is moved from stage to stage, and each element has exactly one owner at a time. A stage that drops an element (`filter`, `take`) must free it, which the operator does through the element's normal ownership rules.
* **Errors are values.** A producer or operator returns `void ! E`; the terminal operation returns `void ! E` to its caller. There are no exceptions and no `onError` callbacks.

## 2. Cost Model

Following `vision.md` Rules 1 and 3, a flow costs what its stages ask for, and every asynchronous boundary has a name:

| Stage kind | Examples | Cost |
| ---------- | -------- | ---- |
| **Synchronous** | `map`, `filter`, `take`, `scan`, `distinct`, `flatMap` | Fused into the producer's loop. Each operator is a generic adapter with its lambda inlined (`core/fp.md`), so `filter(…).map(…).collect(…)` compiles to one loop with no allocation and no call per element |
| **Asynchronous** | `buffer(n)`, `merge`, `zip`, `onFiber` | Starts a fiber and a `Channel` of the stated capacity. The capacity is always written |
| **Timed** | `debounce`, `sample`, `window(Duration)`, `timeout`, `ticker` | Uses `Fiber.sleep` from `bestie.lib.concurrency` |

A chain of synchronous operators has a long generated type, like an iterator chain. To store a flow in a field or return one whose type cannot be written, box it: `flow.boxed()` returns `own BoxedFlow<T, E>`, which costs one allocation and one indirect call per element — the same trade as `Closure` (`std-lib/functional.md`).

## 3. Creating Flows

| Function | Produces |
| -------- | -------- |
| `flow(producer)` | Values the producer emits |
| `flowOf(a, b, c)` | The given values |
| `xs.asFlow()` | The elements of any `Iterable<T>` |
| `ch.asFlow()` | Values received from a `Channel<T>` until it closes |
| `ticker(period: Duration)` | An `Instant` every `period`, forever |
| `cursor.asFlow()` | Rows from a `bestie.api.db` cursor, `Flow<Row, DbError>` |
| `reader.lines()` | Lines from a `TextReader`, `Flow<str, IoError>` |

## 4. Operators

| Group | Operators |
| ----- | --------- |
| Transform | `map`, `mapNotAbsent` (drops `absent` results), `flatMap`, `scan`, `chunked(n)` |
| Filter | `filter`, `take(n)`, `takeWhile`, `drop(n)`, `distinct`, `distinctBy` |
| Combine | `merge(a, b, …)`, `zip(a, b)`, `concat(a, b)`, `combineLatest(a, b)` |
| Time | `debounce(d)`, `sample(d)`, `window(d)`, `timeout(d)`, `delay(d)` |
| Concurrency | `buffer(n)`, `onFiber()`, `parallelMap(workers, f)` — preserves order; `unorderedMap` does not |
| Errors | `retry(times, backoff)`, `catchError(f)` (replace a failure with a fallback flow), `mapError(f)` |
| Side effects | `onEach`, `onComplete` |

Terminal operations:

| Operation | Returns |
| --------- | ------- |
| `collect(f)` | `void ! E` |
| `toList()` | `own List<T> ! E` — allocates, by name |
| `first()` | `T ? ! E` — absent if the flow is empty |
| `fold(init, f)` | `R ! E` |
| `count()` | `int ! E` |
| `launch()` | `Fiber` collecting in the background; cancel it to stop |

Every operator is an extension function over the `Flow` protocol. Third-party operators are written the same way and fuse the same way.

## 5. Hot Flows — `Broadcast<T>`

A `Flow` is cold. Values that exist whether or not anyone listens — prices, sensor readings, domain events — go through a `Broadcast<T>`:

```bestie
val own prices = Broadcast<Quote>.new(capacity: 256, overflow: Overflow.DropOldest)

prices.publish(q)                        // from any fiber or thread
prices.subscribe()                       // a Flow<Quote, never> for this subscriber
    .sample(Duration.new(100 as Millis))
    .collect(q => ui.show(q))
```

* Each subscriber has its own bounded buffer of `capacity`.
* `overflow` says what happens when a subscriber falls behind: `Suspend` (the publisher waits), `DropOldest`, `DropNewest`, or `Fail` (that subscriber's flow fails with `StreamError.Overflow`). There is no unbounded default.
* `T` must be copyable, since every subscriber receives each value. To share an owned object, publish a `ptr` and state its lifetime, or publish a copyable summary.

## 6. With `bestie/web`

```bestie
@Controller("/quotes")
class QuoteController {
    val ref prices: Broadcast<Quote>

    @Get("/live")
    fun live(): own SseStream {
        return SseStream.from(prices.subscribe().map(q => json.serialize(q)))
    }
}
```

When the client disconnects, the response stops collecting, the flow receives `Stopped`, and the subscription is released. Nothing leaks.

## 7. Cancellation

Collecting a flow runs in the caller's fiber. Cancelling that fiber, or a `CancellationToken` passed to `collect`, stops every stage: synchronous stages see `Stopped` from `emit`, asynchronous stages have their fibers cancelled and their channels closed. A stage's owned elements in flight are freed by their channels' normal close rules.

## 8. Non-Goals

* No push-based `Publisher`/`Subscriber`/`request(n)` protocol — fibers make it unnecessary
* No hidden thread pools or schedulers; every fiber is started by a named operator
* No unbounded buffers
* No exceptions or error callbacks; failures are `E`
* Not a replacement for `Iterable`: a finite in-memory sequence uses `bestie.lib.functional`

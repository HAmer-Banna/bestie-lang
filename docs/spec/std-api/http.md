# bestie.api.http — HTTP API

This document defines the **Bestie Standard HTTP API (`bestie.api.http`)**.

`bestie.api.http` provides a **correct, explicit, and minimal HTTP protocol implementation** built on `bestie.api.network` and the stream protocols of `bestie.api.io`.

It is **not** a web framework.

---

## 1. Scope and Non-Goals

### 1.1 What This API Provides

* HTTP/1.1, HTTP/2, and HTTP/3 request and response modeling
* A single request/response model shared across all versions
* Explicit protocol version selection and negotiation
* Header parsing and serialization, including HPACK and QPACK compression
* Connection and stream multiplexing (HTTP/2 and HTTP/3)
* Bodies as ordinary `InputStream`s

### 1.2 What This API Does *Not* Provide

* Routing, controllers, middleware pipelines
* Dependency injection
* Authentication frameworks
* REST or GraphQL abstractions
* Its own body abstraction — a body is an `InputStream` (§6)
* TLS — see §10

These belong in **std-framework** or external libraries.

---

## 2. Design Principles

1. **Protocol correctness first** — the model represents what HTTP actually allows, including repeated headers
2. **No magic defaults**
3. **Explicit request lifecycle**
4. **Streaming, not buffering**
5. **Composable, not opinionated**
6. **Reuse the layers below** — no second stream type, no second buffer type, no second address type

---

## 3. Namespacing

```text
bestie.api.http
```

---

## 4. Class Kinds and Ownership Rationale

| Type | Kind | Why |
| ---- | ---- | --- |
| `HttpVersion` | `enum` | A closed set of three wire protocols. |
| `HttpMethod` | `enum` | A closed set. Extension methods are not supported; see §5.2. |
| `HttpStatus` | newtype | `uint16 in 100..=599` — a status outside that range is not a status, and the range constraint checks it once at construction (`core/lang.md` §6.2). |
| `Headers` | `class` | Owns its storage and is mutated while a message is built. Identity matters — two header sets with the same fields are still separate objects. |
| `HttpRequest` | `class` | Owns its headers and body. The body is a **stream consumed once**, which is state; a `data class` holding one would claim a deep immutability it does not have. |
| `HttpResponse` | `class` | Same. |
| `HttpServer` | `class` | Owns a bound listener. |
| `HttpClient` | `class` | Owns connection state and configuration. |

**Why `HttpRequest` is not a `data class`.** Structural equality on an HTTP message is meaningless — two requests carrying identical bytes are still different requests, arriving on different connections at different times. And a `data class` must be deeply immutable, which a streaming body cannot be: reading it changes it. Identity is the honest model.

---

## 5. Messages

### 5.1 `HttpVersion`

```bestie
enum HttpVersion {
    Http11,
    Http2,
    Http3
}
```

The same `HttpRequest` and `HttpResponse` types are used for every version. The version determines **wire encoding and connection behavior only** — it never changes the semantic model the caller sees.

### 5.2 `HttpMethod`

```bestie
enum HttpMethod {
    Get, Post, Put, Delete, Patch, Options, Head, Trace, Connect
}
```

The set is closed. HTTP permits arbitrary method tokens, but a closed enum makes `switch` exhaustive and keeps a server from dispatching on a method it has never seen. A protocol that needs its own verbs belongs on top of this API, not inside it.

### 5.3 `HttpStatus`

```bestie
type HttpStatus as uint16 in 100..=599
```

A range-constrained newtype (`core/lang.md` §6.2): checked once where a status is constructed, then trusted everywhere. Common values are `const`s in the package namespace:

```bestie
const OK          : HttpStatus = 200 as HttpStatus
const NOT_FOUND   : HttpStatus = 404 as HttpStatus
const SERVER_ERROR: HttpStatus = 500 as HttpStatus
```

### 5.4 `HttpRequest` and `HttpResponse`

```bestie
class HttpRequest {
    val version: HttpVersion
    val method:  HttpMethod
    val path:    str
    val own headers: Headers
    val own body:    InputStream ?     // absent when the message has no body
}

class HttpResponse {
    val version: HttpVersion
    val status:  HttpStatus
    val own headers: Headers
    val own body:    InputStream ?
}
```

* `version` on a request is the **preferred** version; the effective one is reported on the response (§9.1).
* Pseudo-headers (`:method`, `:path`, `:authority`, `:scheme`) used by HTTP/2 and HTTP/3 are derived from these fields — callers never set them directly, and setting one through `Headers` is `HttpError.InvalidHeader`.
* Both own their headers and body, so `freeDeep()` releases the whole message.

---

## 6. Bodies Are Streams

There is **no `HttpBody` type**. A body is an `InputStream` from `bestie.api.io`:

```bestie
val own body = req.body else { return emptyResponse() }
val own text = TextReader.new(move body, Encoding.Utf8)
val json = try text.readAll()
```

A dedicated `HttpBody` protocol would be `InputStream` under another name — one `read` method over a byte buffer. Reusing `InputStream` instead means every tool that already works on streams — `copy`, `BufferedReader`, `TextReader` — works on an HTTP body with no adapter:

```bestie
// Stream a response body straight to a file. No HTTP-specific code.
fun save(resp: ptr<HttpResponse>, out: ptr<File>): int ! IoError {
    val own body = resp.val.body else { return 0 }
    return try copy(body.address(), out)
}
```

`body` is `InputStream ?` because a `GET` request and a `204 No Content` response genuinely have none. Absence is the type's job, not a zero-length stream pretending.

**Backpressure** surfaces the same way it does on any stream: a `read` blocks, and on HTTP/2 or HTTP/3 it blocks until the flow-control window opens (§9.2).

---

## 7. Headers

```bestie
class Headers {
    fun get(name: str): str ?           // first value for the field
    fun getAll(name: str): slice<str>   // every value, in wire order
    fun set(name: str, value: str): void       // replaces all existing values
    fun add(name: str, value: str): void       // appends another value
    fun remove(name: str): void
    fun contains(name: str): bool
    fun names(): slice<str>
    fun free(): void
}
```

An empty set is `Headers.new()`; one parsed from the wire is produced by `accept()` or `send()`.

**Headers are not a `map<str, str>`.** HTTP permits a field to appear more than once — `Set-Cookie` routinely does, and `Accept-Encoding` and `Via` may — so a map keyed by name silently discards data. This API's first principle is protocol correctness, and a model that cannot represent a valid message fails it.

Rules:

* Field names are matched **case-insensitively** (`Content-Type` and `content-type` are one field) and preserved in the case they were written for HTTP/1.1 serialization. HTTP/2 and HTTP/3 lower-case them on the wire, as those protocols require.
* `get` returns the first value; `getAll` returns all of them. A caller that treats a possibly-repeated field as single is making that choice visibly.
* `set` replaces; `add` appends. There is no method that "sets or appends depending on the field", because that would require this API to know which fields repeat.
* Values are not parsed. `Content-Type` is a `str`, not a structured media type — parsing belongs above this layer.

---

## 8. Server and Client

### 8.1 `HttpServer`

```bestie
class HttpServer {
    fun accept(): own HttpRequest ! HttpError
    fun localAddress(): SocketAddress
    fun close(): void
}

fun bind(addr: SocketAddress): own HttpServer ! HttpError
```

An accepted request carries the means to answer it, so responding is a **consuming method on the request** (`core/oop.md` §9.1.1):

```bestie
class HttpRequest {
    // ... fields as in 5.4 ...
    fun respond(own this, own resp: HttpResponse): void ! HttpError
}
```

`accept()` blocks and returns one request. There is **no `serve(handler)` loop**: a callback-driven server hides where concurrency happens, and this API's job is to leave that decision with the caller. The loop is short, and you can see the fiber:

```bestie
import bestie.lib.concurrency.fiber

fun run(addr: SocketAddress): void ! HttpError {
    val own server = try bind(addr)
    defer server.close()

    while (true) {
        val own req = try server.accept()

        // The request is moved into the fiber, which owns it from here.
        fiber.new([move req]() => {
            val own resp = handle(req.address())
            req.respond(move resp) catch |e| { logFailure(e) }
        })
    }
}
```

Putting `respond` on the request rather than the server is what makes the fiber self-contained: the worker needs nothing but the request it was handed, so no shared server handle crosses the boundary and there is nothing to synchronise.

A request that is accepted and never answered is an **unresolved ownership obligation**, so forgetting to respond is a compile-time error rather than a client waiting until it times out.

### 8.2 `HttpClient`

```bestie
class HttpClient {
    fun send(own req: HttpRequest): own HttpResponse ! HttpError
    fun close(): void
}

fun client(preferred: HttpVersion): own HttpClient ! HttpError
```

```bestie
fun fetch(host: str): void ! (HttpError | NetError) {
    val own addrs = try resolve(host, try (443 as Port))
    defer addrs.freeDeep()

    val own http = try client(HttpVersion.Http2)
    defer http.close()

    val own headers = Headers.new()
    headers.set("host", host)
    headers.set("accept", "text/html")

    val own req = HttpRequest.new(
        version: HttpVersion.Http2,
        method:  HttpMethod.Get,
        path:    "/index.html",
        headers: move headers,
        body:    absent
    )

    val own resp = try http.send(move req)
    defer resp.freeDeep()

    println("status ${resp.status}, via ${resp.version}")
}
```

Rules:

* `send` consumes the request — a request is sent once
* No implicit connection pooling, no retries, no redirect following. A `301` is returned to you as a `301`.
* No implicit name resolution: the address comes from `bestie.api.network.resolve` (§5.3 there), so a client that never resolves performs no DNS

---

## 9. Protocol Versions

HTTP/1.1, HTTP/2, and HTTP/3 share the same request/response model but differ in wire encoding and connection behavior. The version is always **explicit and observable** — there is no hidden upgrade.

### 9.1 Version Negotiation

* The client offers its preferred version; the effective version is reported on `HttpResponse.version`.
* **HTTP/2** is negotiated over TLS via **ALPN** (`h2`). The ALPN exchange happens in the TLS layer (§10) — this API only consumes the negotiated result.
* **HTTP/3** runs over **QUIC** and is discovered via explicit configuration or an `Alt-Svc` advertisement; it is **never** silently substituted for an HTTP/2 or HTTP/1.1 connection.
* If a peer cannot satisfy the preferred version, negotiation falls back deterministically: HTTP/3 → HTTP/2 → HTTP/1.1. Fallback is reported on the response, never hidden.
* No protocol is attempted that the caller did not enable.

### 9.2 HTTP/2

HTTP/2 multiplexes many requests over a single connection.

Properties:

* **Multiplexed streams** — concurrent requests share one connection with independent stream IDs
* **HPACK** header compression
* **Binary framing** instead of text framing
* **Flow control** at both connection and stream level
* Runs over TLS with ALPN `h2` (cleartext `h2c` is **not** supported)

Rules:

* Each `HttpRequest` maps to exactly one stream; the body streams over `DATA` frames
* Server push is **not** offered — it is deprecated and adds implicit behavior this API rejects
* Stream and connection flow-control windows are honored; backpressure surfaces as a blocking `read` on the body stream
* Head-of-line blocking at the TCP layer is inherent to HTTP/2 and is **not** worked around here

### 9.3 HTTP/3

HTTP/3 runs over **QUIC**, a UDP-based transport with built-in TLS 1.3.

Properties:

* **QUIC transport** — independent streams without TCP head-of-line blocking
* **QPACK** header compression
* **Connection migration** support, provided by the QUIC layer
* **0-RTT** resumption is available but **opt-in**, because it weakens replay guarantees

Rules:

* HTTP/3 builds on the QUIC primitives in `bestie.api.network`, not on the TCP path used by HTTP/1.1 and HTTP/2
* TLS 1.3 is mandatory and integral to QUIC; there is no cleartext HTTP/3
* Each request maps to a bidirectional QUIC stream; stream loss does not stall unrelated streams
* 0-RTT must be explicitly requested and is forbidden for non-idempotent methods unless the caller opts in

---

## 10. TLS

TLS is **not** part of this API. It belongs in `bestie.api.tls` or an external package.

ALPN negotiation (selecting `h2` vs `http/1.1`) and the TLS 1.3 handshake required by QUIC/HTTP/3 are performed by the TLS and QUIC layers. This API consumes the negotiated protocol; it does not implement the handshake.

---

## 11. Error Model

```bestie
errors HttpError {
    MalformedMessage,      // the peer sent something that is not valid HTTP
    InvalidHeader,         // includes attempting to set a pseudo-header directly
    HeadersTooLarge,
    UnsupportedVersion,
    VersionNegotiationFailed,
    StreamReset,           // HTTP/2 or HTTP/3 stream terminated by the peer
    ConnectionClosed,
    Timeout
}
```

Transport failures surface as `NetError` from `bestie.api.network`; body read and write failures surface as `ReadError` / `WriteError` from `bestie.api.io`. Error sets compose (`core/exceptions.md` §3.1), so a handler declaring `! (HttpError | NetError | IoError)` propagates all three with `try` and no conversion.

Rules:

* Protocol errors are explicit and typed
* A malformed message is never partially delivered — `accept()` fails rather than yielding a half-parsed request
* No silent recovery, no automatic retry, no redirect following

---

## 12. Relationship to Other Layers

| Concern | Package |
| ------- | ------- |
| Sockets, addresses, DNS | `bestie.api.network` |
| Body streams, buffering, text decoding | `bestie.api.io` |
| TLS and ALPN | `bestie.api.tls` (external) |
| Routing, controllers, middleware | `bestie.framework.web` |

`bestie.api.http` provides **raw HTTP mechanics**. REST frameworks, GraphQL servers, RPC frameworks, and WebSocket implementations are built on top of it, not inside it.

---

## 13. Summary

`bestie.api.http` is protocol-correct, explicit, minimal, and framework-agnostic:

* **Headers model what HTTP allows.** Repeated fields are representable, matching is case-insensitive, and no valid message is inexpressible.
* **A body is an `InputStream`.** No second stream abstraction, so `copy`, `BufferedReader`, and `TextReader` all work on HTTP bodies unchanged.
* **Messages are `class`es with identity.** Two requests with identical bytes are different requests, and a streaming body is state a `data class` could not honestly hold.
* **The server loop is yours.** `accept()` returns one request; there is no callback that hides where concurrency happens.
* **An unanswered request will not compile.** `respond` consumes both messages, so an accepted request is an ownership obligation.
* **One model across three versions.** HTTP/1.1, HTTP/2, and HTTP/3 differ in wire encoding only, and the effective version is always reported.

It is the **foundation**, not the final product.

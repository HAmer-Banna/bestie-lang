# bestie.api.http — HTTP API

This document defines the **Bestie Standard HTTP API (`bestie.api.http`)**.

`bestie.api.http` provides a **correct, explicit, and minimal HTTP protocol implementation** built on top of `bestie.api.network`.

It is **not** a web framework.

---

## 1. Scope and Non-Goals

### 1.1 What This API Provides

* HTTP/1.1, HTTP/2, and HTTP/3 request and response modeling
* A single request/response model shared across all versions
* Explicit protocol version selection and negotiation
* Header parsing and serialization (including HPACK and QPACK compression)
* Connection and stream multiplexing (HTTP/2 and HTTP/3)
* Explicit body streaming

---

### 1.2 What This API Does *Not* Provide

* Routing
* Controllers
* Middleware pipelines
* Dependency injection
* Authentication frameworks
* REST or GraphQL abstractions

These belong in **std-framework** or external libraries.

---

## 2. Design Principles

1. **Protocol correctness first**
2. **No magic defaults**
3. **Explicit request lifecycle**
4. **Streaming, not buffering**
5. **Composable, not opinionated**

---

## 3. Namespacing

```text
bestie.api.http
```

---

## 4. Core Types

### 4.1 `HttpVersion`

```bestie
enum HttpVersion {
    Http11, Http2, Http3
}
```

The same `HttpRequest` and `HttpResponse` types are used for every version. The version determines **wire encoding and connection behavior only** — it never changes the semantic request/response model the caller sees.

---

### 4.2 `HttpMethod`

```bestie
enum HttpMethod {
    GET, POST, PUT, DELETE, PATCH, OPTIONS, HEAD
}
```

---

### 4.3 `HttpRequest`

```bestie
class HttpRequest {
    version: HttpVersion
    method: HttpMethod
    path: str
    headers: map<str, str>
    body: HttpBody
}
```

* `version` is the **preferred** version. The effective version may differ after negotiation (see section 8).
* Pseudo-headers (`:method`, `:path`, `:authority`, `:scheme`) used by HTTP/2 and HTTP/3 are derived from these fields — callers never set them directly.

---

### 4.4 `HttpResponse`

```bestie
class HttpResponse {
    version: HttpVersion
    status: int
    headers: map<str, str>
    body: HttpBody
}
```

* `version` reflects the version actually used on the wire.

---

## 5. Body Handling

### 5.1 `HttpBody`

```bestie
protocol HttpBody {
    fun read(buffer: ptr<ByteBuffer>): int
}
```

Properties:

* Streaming by design
* No implicit buffering
* No implicit content-length

---

## 6. HTTP Server

### 6.1 `HttpServer`

```bestie
class HttpServer {
    fun serve(handler: (HttpRequest) -> HttpResponse): void
}
```

Rules:

* Handler is synchronous
* No hidden concurrency
* Ownership is explicit
* Errors propagate explicitly

---

### 6.2 Binding

```bestie
fun bind(addr: SocketAddress): HttpServer
```

---

## 7. HTTP Client

### 7.1 `HttpClient`

```bestie
class HttpClient {
    fun send(req: HttpRequest): HttpResponse
}
```

Rules:

* No implicit connection pooling
* No retries
* No redirects unless explicitly handled

---

## 8. Protocol Versions

HTTP/1.1, HTTP/2, and HTTP/3 share the same request/response model but differ in wire encoding and connection behavior. The version is always **explicit and observable** — there is no hidden upgrade.

### 8.1 Version Negotiation

```bestie
class HttpClient {
    fun send(req: HttpRequest): HttpResponse
    fun preferred(version: HttpVersion): HttpClient
}
```

Rules:

* The client offers `req.version` as its preferred version; the effective version is reported on `HttpResponse.version`.
* **HTTP/2** is negotiated over TLS via **ALPN** (`h2`). The ALPN exchange happens in the TLS layer (see section 9) — this API only consumes the negotiated result.
* **HTTP/3** runs over **QUIC** (see `bestie.api.network`) and is discovered via explicit configuration or an `Alt-Svc` advertisement; it is **never** silently substituted for an HTTP/2 or HTTP/1.1 connection.
* If a peer cannot satisfy the preferred version, negotiation falls back deterministically: HTTP/3 → HTTP/2 → HTTP/1.1. Fallback is reported, never hidden.
* No protocol is attempted that the caller did not enable.

---

### 8.2 HTTP/2

HTTP/2 multiplexes many requests over a single connection.

Properties:

* **Multiplexed streams** — concurrent requests share one connection with independent stream IDs
* **HPACK** header compression
* **Binary framing** instead of text framing
* **Flow control** at both connection and stream level
* Runs over TLS with ALPN `h2` (cleartext `h2c` is **not** supported)

Rules:

* Each `HttpRequest` maps to exactly one stream; the `HttpBody` streams over `DATA` frames
* Server push is **not** offered — it is deprecated and adds implicit behavior this API rejects
* Stream and connection flow-control windows are honored; backpressure surfaces through `HttpBody.read`
* Head-of-line blocking at the TCP layer is inherent to HTTP/2 and is **not** worked around here

---

### 8.3 HTTP/3

HTTP/3 runs over **QUIC**, a UDP-based transport with built-in TLS 1.3.

Properties:

* **QUIC transport** — independent streams without TCP head-of-line blocking
* **QPACK** header compression
* **Connection migration** support (provided by the QUIC layer)
* **0-RTT** resumption is available but **opt-in**, because it weakens replay guarantees

Rules:

* HTTP/3 builds on the QUIC primitives in `bestie.api.network`, not on the TCP path used by HTTP/1.1 and HTTP/2
* TLS 1.3 is mandatory and integral to QUIC; there is no cleartext HTTP/3
* Each request maps to a bidirectional QUIC stream; stream loss does not stall unrelated streams
* 0-RTT must be explicitly requested and is forbidden for non-idempotent methods unless the caller opts in

---

## 9. TLS

TLS is **not** part of this API.

Possible locations:

* `bestie.api.tls`
* External libraries

This keeps HTTP logic clean and testable.

ALPN negotiation (selecting `h2` vs `http/1.1`) and the TLS 1.3 handshake required by QUIC/HTTP/3 are performed by the TLS and QUIC layers. This API consumes the negotiated protocol; it does not implement the handshake.

---

## 10. Error Model

* Protocol errors are explicit
* Invalid requests are surfaced
* No silent recovery

---

## 11. Relationship to Frameworks

`bestie.api.http` provides **raw HTTP mechanics**.

Examples built on top:

* REST frameworks
* GraphQL servers
* RPC frameworks
* WebSocket frameworks

---

## 12. Summary

`bestie.api.http` is:

* Protocol-correct
* Explicit
* Minimal
* Multi-version (HTTP/1.1, HTTP/2, HTTP/3) over one request/response model
* Framework-agnostic

It is the **foundation**, not the final product.

This document is **finalized**.

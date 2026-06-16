# bestie.api.http — HTTP API

This document defines the **Bestie Standard HTTP API (`bestie.api.http`)**.

`bestie.api.http` provides a **correct, explicit, and minimal HTTP protocol implementation** built on top of `bestie.api.network`.

It is **not** a web framework.

---

## 1. Scope and Non-Goals

### 1.1 What This API Provides

* HTTP/1.1 request and response modeling
* Header parsing and serialization
* Connection handling
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

### 4.1 `HttpMethod`

```bestie
enum HttpMethod {
    GET, POST, PUT, DELETE, PATCH, OPTIONS, HEAD
}
```

---

### 4.2 `HttpRequest`

```bestie
class HttpRequest {
    method: HttpMethod
    path: str
    headers: map<str, str>
    body: HttpBody
}
```

---

### 4.3 `HttpResponse`

```bestie
class HttpResponse {
    status: int
    headers: map<str, str>
    body: HttpBody
}
```

---

## 5. Body Handling

### 5.1 `HttpBody`

```bestie
protocol HttpBody {
    fun read(buffer: ref ByteBuffer): int
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
    fun serve(handler: fun(HttpRequest): HttpResponse): void
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

## 8. TLS

TLS is **not** part of this API.

Possible locations:

* `bestie.api.tls`
* External libraries

This keeps HTTP logic clean and testable.

---

## 9. Error Model

* Protocol errors are explicit
* Invalid requests are surfaced
* No silent recovery

---

## 10. Relationship to Frameworks

`bestie.api.http` provides **raw HTTP mechanics**.

Examples built on top:

* REST frameworks
* GraphQL servers
* RPC frameworks
* WebSocket frameworks

---

## 11. Summary

`bestie.api.http` is:

* Protocol-correct
* Explicit
* Minimal
* Framework-agnostic

It is the **foundation**, not the final product.

This document is **finalized**.

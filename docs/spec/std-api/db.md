# bestie.api.db — Database API

This document defines the **Bestie Standard Database API (`bestie.api.db`)**.

`bestie.api.db` provides **low-level, explicit access to relational stores**. It is the layer an ORM is built on, not an ORM.

---

## 1. Scope and Non-Goals

### 1.1 What This API Provides

* Connections to database servers
* Parameterized query execution
* Typed column values, including SQL `NULL`
* Streaming result cursors
* Explicit transactions that cannot be silently abandoned
* Prepared statements

### 1.2 What This API Does *Not* Provide

* ORM abstractions (see `bestie.framework.orm`)
* ActiveRecord patterns
* Caching layers
* Async query frameworks
* Migration DSLs
* A SQL parser or dialect abstraction — the query string is passed to the server as written

---

## 2. Design Principles

1. **Connections are explicit**
2. **Transactions are explicit, and the compiler enforces that they end**
3. **Values are typed** — a database column is not a string
4. **Results stream** — a result set is not materialized unless you ask
5. **Parameters are bound, never interpolated**
6. **Errors are typed and explicit**
7. **Classes for stateful resources, enums for closed value sets**

---

## 3. Namespacing

```text
bestie.api.db
```

---

## 4. Class Kinds and Ownership Rationale

| Type | Kind | Why |
| ---- | ---- | --- |
| `DbValue` | `enum` | A column holds one of a closed set of types. Payload variants carry the value inline, and exhaustive `switch` forces every caller to handle every column type. |
| `DbConnection` | `class` | Owns a socket and server-side session state. Identity matters — two connections to the same database are different sessions. |
| `DbTransaction` | `class` | Owns the in-progress transaction. Identity is essential: committing "a" transaction rather than *this* one would be a correctness bug. |
| `DbCursor` | `class` | Streaming position over a result set. Mutable state by definition. |
| `Row` | `class` | Owns its column values for the current position. Identity distinguishes it from the next row. |
| `PreparedStatement` | `class` | Owns a server-side handle that must be released. |
| `Isolation` | `enum` | A closed set of isolation levels. |

Every resource here is an `own` value. `DbConnection`, `DbCursor` and `PreparedStatement` are discharged with `close()`; `DbTransaction` is discharged by `commit()` or `rollback()` (§8), which is what makes an abandoned transaction a **compile-time error** rather than a leaked lock.

---

## 5. Values

### 5.1 `DbValue`

```bestie
enum DbValue {
    Integer(int64),
    Real(float64),
    Text(str),
    Bool(bool),
    Blob(own array<byte>)
}
```

A column value is **typed**. An earlier shape returned `map<str, str>` — every column stringified — which loses the server's type information, forces every caller to re-parse, and cannot represent a blob at all.

`Blob` carries an `own` payload, so a `DbValue` holding one is **move-only** (`core/memory.md` §4.3). The other four variants are freely copyable.

### 5.2 SQL `NULL` is absence

`NULL` is not a `DbValue` variant. A nullable column is `DbValue ?`:

```bestie
val name : DbValue ? = try row.get("name")
```

SQL `NULL` means *this column has no value*, which is exactly what `T ?` means in Bestie (`core/types.md` §8.3). Adding a `DbValue.Null` variant would create a second absence vocabulary alongside the one core already has, and callers would have to handle both.

---

## 6. Connections

```bestie
class DbConnection {
    fun execute(sql: str, params: slice<DbValue>): own DbCursor ! DbError
    fun executeUpdate(sql: str, params: slice<DbValue>): int ! DbError
    fun prepare(sql: str): own PreparedStatement ! DbError
    fun begin(level: Isolation): own DbTransaction ! DbError
    fun close(): void
}

fun connect(url: str): own DbConnection ! DbError
```

* `execute` returns a streaming cursor; `executeUpdate` returns the number of affected rows for statements that produce none.
* `params` is a `slice<DbValue>` — a borrowed view over a caller-owned array, so binding parameters allocates nothing.
* A `DbConnection` is **not** thread-safe. Move it into the thread or fiber that uses it, or guard it with a `Lock`.

### 6.1 Parameters are bound, never interpolated

Every execution path takes parameters separately from the SQL text. There is no overload that takes a query alone with values embedded:

```bestie
var args : array<DbValue>[1] = array<DbValue>[1].fill(DbValue.Integer(0))
args[0] = DbValue.Text(userInput)

val own cur = try conn.execute("SELECT id, email FROM users WHERE name = ?", args[..])
defer cur.close()
```

Interpolating a value into the SQL string is possible — nothing can stop string concatenation — but this API never makes it the shorter path. `params` is a required argument, so a query with no parameters still passes an empty slice:

```bestie
val noArgs : array<DbValue>[0]
val own cur = try conn.execute("SELECT count(*) FROM users", noArgs[..])
```

That is the design decision: binding is never more work than interpolating, so the safe form is also the convenient one.

---

## 7. Results

### 7.1 `DbCursor` — results stream

```bestie
class DbCursor {
    fun next(): own Row ? ! DbError
    fun columns(): slice<str>
    fun close(): void
}
```

`next()` returns **a row, a clean end of results, or a failure** — three distinct outcomes, which is the `T ? ! E` shape (`core/types.md` §8.4). That makes a cursor a fallible iterator, consumed with `try for` (`core/lang.md` §13):

```bestie
fun listUsers(conn: ptr<DbConnection>): void ! DbError {
    val noArgs : array<DbValue>[0]
    val own cur = try conn.val.execute("SELECT id, email FROM users", noArgs[..])
    defer cur.close()

    try for (row in cur) {
        defer row.freeDeep()
        val id    = try row.get("id")    else { continue }
        val email = try row.get("email") else { continue }
        println("${id} ${email}")
    }
}
```

A result set is **never** materialized into a list on your behalf. A query returning ten million rows streams ten million times through that loop and holds one row at a time. To materialize deliberately, collect the cursor yourself.

### 7.2 `Row`

```bestie
class Row {
    fun get(column: str): DbValue ? ! DbError
    fun getAt(index: int): DbValue ? ! DbError
    fun columnCount(): int
}
```

`get` has the same three outcomes for a different reason: the value, `NULL` (absent), or `DbError.NoSuchColumn` (a failure — asking for a column the query did not select is a bug, not a missing value).

Reading a typed value is an ordinary exhaustive `switch`:

```bestie
val v = try row.get("age") else { return }

switch (v) {
    case DbValue.Integer(val n) => useAge(n)
    case DbValue.Text(val s)    => useAge(try s.toInt())
    case _                      => return DbError.TypeMismatch
}
```

---

## 8. Transactions

```bestie
enum Isolation {
    ReadUncommitted,
    ReadCommitted,
    RepeatableRead,
    Serializable
}

class DbTransaction {
    fun execute(sql: str, params: slice<DbValue>): own DbCursor ! DbError
    fun executeUpdate(sql: str, params: slice<DbValue>): int ! DbError

    fun commit(own this): void ! DbError
    fun rollback(own this): void ! DbError
}
```

### 8.1 An abandoned transaction is a compile-time error

`commit` and `rollback` **consume** the transaction (`core/oop.md` §9.1.1). Because `begin()` returns an `own DbTransaction`, that ownership obligation must be discharged exactly once before the binding leaves scope (`core/memory.md` §7.3) — and the only two things that discharge it are committing and rolling back.

```bestie
fun transfer(conn: ptr<DbConnection>, from: int64, to: int64, amount: int64): void ! DbError {
    var debit : array<DbValue>[2] = array<DbValue>[2].fill(DbValue.Integer(0))
    debit[0] = DbValue.Integer(amount)
    debit[1] = DbValue.Integer(from)

    var credit : array<DbValue>[2] = array<DbValue>[2].fill(DbValue.Integer(0))
    credit[0] = DbValue.Integer(amount)
    credit[1] = DbValue.Integer(to)

    val own tx = try conn.val.begin(Isolation.Serializable)

    tx.executeUpdate("UPDATE accounts SET balance = balance - ? WHERE id = ?", debit[..])
        catch |e| { try tx.rollback(); return e }

    tx.executeUpdate("UPDATE accounts SET balance = balance + ? WHERE id = ?", credit[..])
        catch |e| { try tx.rollback(); return e }

    try tx.commit()
}
```

Forgetting the final `commit()` is not a runtime bug that shows up as a held lock in production — it is:

```
error: ownership of 'tx' is not discharged before scope exits
```

This is the single strongest argument for explicit ownership in this API. Languages with garbage collection cannot make this check; languages with RAII make it by silently rolling back at scope exit, which hides the mistake instead of reporting it. Bestie refuses to guess what an unfinished transaction meant and refuses to compile it.

`defer tx.rollback()` is **not** the idiom here — `defer` cannot consume `this`, and a deferred rollback after a successful commit would be a double discharge, which the compiler also rejects.

---

## 9. Prepared Statements

```bestie
class PreparedStatement {
    fun execute(params: slice<DbValue>): own DbCursor ! DbError
    fun executeUpdate(params: slice<DbValue>): int ! DbError
    fun close(): void
}
```

* The statement owns a server-side handle and must be closed
* It is tied to the connection that prepared it. Using it after that connection closes is `DbError.Closed`, not undefined behavior
* Re-executing with different parameters is the point — prepare once outside a loop, execute inside it

---

## 10. Error Model

```bestie
errors DbError {
    ConnectionFailed,
    Closed,             // operation on a closed connection, cursor, or statement
    Timeout,
    SyntaxError,        // the server rejected the SQL
    ConstraintViolation,
    NoSuchColumn,
    TypeMismatch,
    Deadlock,
    PermissionDenied
}
```

Rules:

* No exceptions, no implicit retries — a `Deadlock` is returned to you, and whether to retry is your policy
* No exception is made for "no rows": a query matching nothing yields a cursor whose first `next()` is absent, which is success
* Connection-level failures do not roll back on your behalf. If the server drops the connection mid-transaction, the transaction is gone server-side and `commit()` returns `ConnectionFailed`

---

## 11. Relationship to Other APIs

* `bestie.api.network` — drivers connect over sockets from that package
* `bestie.api.io` — a `Blob` streamed rather than materialized uses the stream protocols
* `bestie.framework.orm` — builds on this package; nothing here knows it exists

Driver implementations for specific servers (PostgreSQL, SQLite, MySQL) are **separate packages**, not part of this API. This document defines the shape every driver presents.

---

## 12. Summary

`bestie.api.db` provides low-level database access with four properties that distinguish it from a thin binding:

* **Values are typed.** `DbValue` is an enum with an exhaustive `switch`, not a stringified map, and SQL `NULL` reuses `T ?` rather than inventing a second absence.
* **Results stream.** A cursor is a fallible iterator (`Row ? ! DbError`), so a large result set never materializes unless you materialize it.
* **Transactions cannot be abandoned.** `commit` and `rollback` consume the transaction, so forgetting both is a compile-time leak error rather than a lock held in production.
* **Parameters are bound.** No execution path takes SQL with values already embedded, and binding is never more work than interpolating would be.

It is a **foundation for higher-level ORM or query frameworks**, and prescribes none of them.

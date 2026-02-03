# std-api.db — Database API

This document defines the **Bestie Standard Database API (`std-api.db`)**.

`std-api.db` provides **low-level, explicit access to relational and key-value stores**.

---

## 1. Scope and Non-Goals

### 1.1 What This API Provides

* Connections to DB servers
* Query execution
* Transactions (explicit)
* Result handling
* Prepared statements

### 1.2 What This API Does *Not* Provide

* ORM abstractions
* ActiveRecord patterns
* Caching layers
* Async query frameworks
* Migration DSLs

---

## 2. Design Principles

1. **Connections are explicit**
2. **Transactions are explicit**
3. **Errors are typed and explicit**
4. **No hidden allocations**
5. **Composable, deterministic behavior**
6. **Classes for stateful resources**
7. **Functions for stateless operations**

---

## 3. Namespacing

```text
std.api.db
```

---

## 4. Core Types

### 4.1 `DbConnection`

```bestie
class DbConnection {
    fun execute(query: str): Result<DbResult, DbError>
    fun beginTransaction(): DbTransaction
    fun close(): void
}
```

---

### 4.2 `DbTransaction`

```bestie
class DbTransaction {
    fun commit(): Result<void, DbError>
    fun rollback(): Result<void, DbError>
}
```

---

### 4.3 `DbResult`

```bestie
class DbResult {
    fun rows(): list<map<str, str>>
    fun rowCount(): int
}
```

---

## 5. Queries

```bestie
fun executeQuery(conn: DbConnection, query: str): Result<DbResult, DbError>
```

* Returns explicit typed errors
* Ownership of results is clear
* No side effects outside explicit connection

---

## 6. Prepared Statements

```bestie
class PreparedStatement {
    fun execute(params: list<str>): Result<DbResult, DbError>
    fun close(): void
}
```

* State tied to connection
* Ownership explicit
* Deterministic cleanup

---

## 7. Error Model

* `DbError` types
* Typed
* Explicit handling required
* No exceptions or implicit retries

---

## 8. Transactions

* `DbTransaction` enforces **explicit commit/rollback**
* Scoped lifetimes recommended
* Ownership rules apply

---

## 9. Summary

`std-api.db` provides:

* Low-level DB access
* Explicit connections and transactions
* Typed, deterministic error handling
* Minimal abstraction layer

It is intended as a **foundation for higher-level ORM or query frameworks**, but **does not prescribe them**.

This document is **finalized**.

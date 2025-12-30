# std-lib.format — Structured Data Formats

This document defines **Bestie Standard Formatting Library (`std-lib.format`)**.

`std-lib.format` provides **parsing and serialization** for common structured data formats used for:

* Configuration
* Interchange
* Persistence
* Tooling

It is part of **std-lib**, not `std-api`, because:

* These formats are **pure data**
* They do not depend on OS or platform facilities
* They are safe, deterministic, and portable

---

## 1. Design Principles

1. **Uniform API across formats**
2. **No reflection**
3. **No implicit schema inference**
4. **Explicit errors**
5. **Composable with user-defined types**

If two formats do similar things, **they behave the same**.

---

## 2. Supported Formats

Initial formats:

* JSON
* XML
* CSV
* TOML
* YAML

Each format lives in its own namespace:

```text
bestie.lib.format.json
bestie.lib.format.xml
bestie.lib.format.csv
bestie.lib.format.toml
bestie.lib.format.yaml
```

---

## 3. Unified Core Abstractions

### 3.1 `Parser<T>`

```bestie
protocol Parser<T> {
    fun parse(input: str): T | ParseError
}
```

---

### 3.2 `Serializer<T>`

```bestie
protocol Serializer<T> {
    fun serialize(value: T): str
}
```

All formats implement these protocols.

---

## 4. JSON Example

```bestie
import bestie.lib.format.json

own user = json.parse<User>(input)
own text = json.serialize(user)
```

Rules:

* No dynamic typing
* No implicit number coercion
* No silent field dropping

---

## 5. XML Example

```bestie
import bestie.lib.format.xml

own doc = xml.parse<Document>(input)
own output = xml.serialize(doc)
```

Rules:

* Explicit element mapping
* No XPath magic
* Deterministic structure

---

## 6. CSV Example

```bestie
import bestie.lib.format.csv

own rows = csv.parse<list<Row>>(input)
own text = csv.serialize(rows)
```

Rules:

* Header handling is explicit
* No schema guessing
* No implicit type conversion

---

## 7. Type Mapping Rules

User-defined types must be explicitly compatible.

```bestie
data class User {
    id: int
    name: str
}
```

Rules:

* Field names must match
* Missing fields are errors
* Extra fields are errors unless explicitly allowed

---

## 8. Custom Format Support

Users may define custom serializers/parsers:

```bestie
class UserJson : Serializer<User>, Parser<User> {
    fun serialize(u: User): str = ...
    fun parse(s: str): User | ParseError = ...
}
```

No reflection hooks are required.

---

## 9. Error Model

All parsing errors are:

* Typed
* Explicit
* Non-exceptional

```bestie
enum ParseError {
    SyntaxError,
    MissingField,
    InvalidType,
    Overflow
}
```

---

## 10. Intentional Non-Features

This library intentionally avoids:

* Runtime reflection
* Schema auto-generation
* Implicit defaults
* Loose typing
* Partial parsing

Correctness is preferred over convenience.

---

## 11. Summary

`std-lib.format` is:

* Uniform
* Predictable
* Explicit
* Safe

It provides **data interchange**, not data guessing.

---

This document is **finalized**.

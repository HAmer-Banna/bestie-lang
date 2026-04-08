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
6. **Unified codec model, not a forced universal data shape**

If two formats do similar things, **they behave the same**.

---

## 2. Supported Formats

Initial official formats:

* JSON
* XML
* CSV
* TOML
* YAML

Each official codec lives in its own namespace:

```text
bestie.lib.format.json
bestie.lib.format.xml
bestie.lib.format.csv
bestie.lib.format.toml
bestie.lib.format.yaml
```

These namespaces are **official std-lib codecs**, not the only formats Bestie can ever handle.
The standard library ships a curated set; future formats do not require changing `std-lib` itself.

---

## 3. Unified Core Abstractions

### 3.1 `Parser<T>`

```bestie
protocol Parser<T> {
    fun parse(input: str): T ! ParseError
}
```

---

### 3.2 `Serializer<T>`

```bestie
protocol Serializer<T> {
    fun serialize(value: T): str
}
```

All formats use `impl` with these protocols.

The **unified part** of `std-lib.format` is the codec interface:

* `parse<T>(input)`
* `serialize(value)`
* shared error handling
* shared explicitness rules

The unified part is **not** a mandatory in-memory tree such as "everything is `map<str, X>`".
Bestie avoids flattening structurally different formats into one lossy representation.

Examples:

* JSON may map naturally to a user type or a `map`
* CSV naturally maps to rows
* XML naturally maps to an explicit document/tree model
* `.properties` naturally maps to `map<str, str>`

Formats share one codec model while preserving their own structure.

---

## 4. JSON Example

```bestie
import bestie.lib.format.json

val user = try json.parse<User>(input)
val text = json.serialize(user)
```

Rules:

* No dynamic typing
* No implicit number coercion
* No silent field dropping
* Parsing into `map<str, T>` is allowed when the target shape is explicitly requested

---

## 5. XML Example

```bestie
import bestie.lib.format.xml

val doc = try xml.parse<Document>(input)
val output = xml.serialize(doc)
```

Rules:

* Explicit element mapping
* No XPath magic
* Deterministic structure
* XML is not forced into a `map` model

---

## 6. CSV Example

```bestie
import bestie.lib.format.csv

val rows = try csv.parse<list<Row>>(input)
val text = csv.serialize(rows)
```

Rules:

* Header handling is explicit
* No schema guessing
* No implicit type conversion
* CSV remains row-oriented, not object-tree-oriented

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
* Generic container targets such as `map<str, str>` or `list<Row>` are valid only when explicitly requested by the caller

---

## 8. Official vs External Codecs

`bestie.lib.format.*` is reserved for **official standard codecs** maintained as part of Bestie.

External libraries may define additional codecs without waiting for a std-lib update.
They use the same `Parser<T>` / `Serializer<T>` protocols, but live in their own package namespace.

Examples:

```text
acme.format.properties
org.tools.format.ini
company.config.format.hocon
```

This keeps the standard library stable while still allowing the ecosystem to grow.

If a format becomes widely important and broadly stable, it may later be promoted into `bestie.lib.format.*`.

---

## 9. Custom Format Support

Users may define custom serializers/parsers:

```bestie
class UserJson impl Serializer<User>, Parser<User> {
    fun serialize(u: User): str = ...
    fun parse(s: str): User ! ParseError = ...
}
```

No reflection hooks are required.

Custom codecs may target:

* user-defined data classes
* explicit document models
* explicit container shapes such as `map<str, str>` when the format naturally fits them

Example:

```bestie
import acme.format.properties

own props = properties.parse<map<str, str>>(input)
own text = properties.serialize(props)
```

---

## 10. Error Model

All parsing errors are:

* Typed
* Explicit
* Non-exceptional

```bestie
errors ParseError {
    SyntaxError,
    MissingField,
    InvalidType,
    Overflow
}
```

---

## 11. Intentional Non-Features

This library intentionally avoids:

* Runtime reflection
* Schema auto-generation
* Implicit defaults
* Loose typing
* Partial parsing
* A required universal "document as map" representation

Correctness is preferred over convenience.

---

## 12. Summary

`std-lib.format` is:

* Uniform
* Predictable
* Explicit
* Safe
* Open to new codecs through the same protocol model

It provides **data interchange**, not data guessing.

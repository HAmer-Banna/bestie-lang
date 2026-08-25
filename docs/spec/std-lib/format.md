# bestie.lib.format — Structured Data Formats

This document defines **Bestie Standard Formatting Library (`bestie.lib.format`)**.

`bestie.lib.format` provides **parsing and serialization** for common structured data formats used for:

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

The **unified part** of `bestie.lib.format` is the codec interface:

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
* Missing fields are errors — **unless** the field declares a default, which makes it optional on deserialize (see §11.9 for schema evolution)
* Extra fields are errors unless explicitly allowed (opt-in per call, §11.9)
* Generic container targets such as `map<str, str>` or `list<Row>` are valid only when explicitly requested by the caller
* Construction always runs the target type's constructor, so type invariants are enforced (see §11.7)

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

## 11. Serialization and the Memory Model

Serialization captures a **value snapshot** of an object graph; deserialization rebuilds an **owned, independent** graph from that snapshot. Because Bestie's memory model is explicit about ownership and indirection, serialization behavior is defined precisely per field kind — there is no reflection and no guessing.

### 11.1 Field Behavior by Ownership Kind

| Field kind | Serialized? | Deserialized as |
| ---------- | ----------- | --------------- |
| value (`primitive`, `value class`, `data class`, `tuple`, `enum`) | ✅ by value | the same value |
| `own T` | ✅ — the owned content is part of the graph | a **newly allocated** owned `T` (caller owns it) |
| `ref T` | ❌ **not serialized** (transient) | not restored — must be re-attached by the programmer |
| `ptr<T>` | ❌ **not serialized** (transient) | not restored — a raw address has no portable meaning |

Rationale:

* A `ref` field is a non-owning stored handle — on deserialization there is no owner attached, so it cannot be reconstructed automatically.
* A `ptr<T>` is a raw machine address; it is meaningless in another process, another run, or after relocation. It is transient by definition.

### 11.2 Auto-Derivation and Custom Codecs

* A type is **auto-serializable** (compiler-derivable `Serializer<T>` / `Parser<T>`) when every serialized field is itself serializable — i.e. value fields and `own` fields of serializable types. `data class`es are the common case and derive trivially.
* `ref` and `ptr<T>` fields are **skipped**. If a type cannot be validly reconstructed without them, the compiler cannot derive a `Parser` for it — the programmer must supply a **custom `Parser<T>`** (§9) that re-establishes those links after the owned fields are built.
* There is **no `Serializable` marker protocol** — serialization capability is expressed by satisfying `Serializer<T>` / `Parser<T>` (method-bearing), exactly as duplication is expressed by `Copyable` / `DeepCopyable` (`util.md` §7). Capability is always a real method, never an empty tag.

### 11.3 Containers

A container serializes element-by-element, following the element kind from §11.1:

| Container | Serialization |
| --------- | ------------- |
| `list<value T>` / `list<own T>` | every element serialized; deserializes to an **owning** container the caller owns |
| `list<ref T>` | elements are non-owning handles → **not serializable** (a custom codec must decide how to resolve them) |
| `list<ptr<T>>` | raw addresses → **not serializable** |

A deserialized collection always **owns** its elements and its buffer — there is no way to deserialize into a non-owning collection.

### 11.4 Immutability

Immutable data (`data class`, `str`, `const` values) serializes and deserializes cleanly — it is pure value data. Deserialization produces fresh immutable values; no special handling is required, and the result is safe to share across threads under the usual rules.

### 11.5 Relationship to `copy` / `deepCopy`

A serialize → deserialize round-trip is closely related to `deepCopy` (`util.md` §7):

* Both produce a **fully owned, independent** graph of the value-and-`own` portion.
* Both **do not follow `ptr<T>`** — raw pointers are outside the managed graph.
* The difference: `deepCopy` preserves `ref` aliasing (the copy holds the same non-owning handles), whereas serialization **drops** `ref` entirely (there is no stored handle to restore after a round-trip).

### 11.6 Cycles

A pure ownership graph is acyclic: every `own` value has exactly one owner, so owned structure forms a tree (or DAG of values). Cycles can exist only through `ref` / `ptr<T>` — and those are never serialized. Therefore **auto-derived serialization cannot encounter an ownership cycle**, and no cycle-detection machinery is needed. Graphs that are cyclic by intent must use a custom codec that encodes identities explicitly.

### 11.7 Deserialization Safety

Deserialization in Bestie treats input as **untrusted data, never as instructions**. This is the core of how Bestie avoids the deserialization vulnerabilities (gadget chains, remote code execution, forged invariants) that plague reflective serialization systems.

**No code execution, no arbitrary instantiation.**

* `parse<T>(input)` constructs **only `T`** (and the value/`own` types statically reachable from `T`). The payload never names a type to instantiate — the target type is fixed at the call site and known at compile time.
* Deserialization runs **no user-selected code paths** and invokes **no magic lifecycle hooks**. There is no `readObject`-style entry point that untrusted bytes can steer. The only code that runs is the target type's own constructor and the (statically chosen) `Parser<T>`.
* No reflection, no dynamic class loading, no polymorphic "instantiate whatever the stream says" behavior.

**Invariants are always enforced — construction goes through the constructor.**

* A deserialized object is built by calling the type's **normal constructor / factory** with the parsed fields. Deserialization **cannot bypass construction** to populate fields directly.
* Therefore every invariant the constructor enforces holds for deserialized objects exactly as for in-program ones. Untrusted input cannot forge an object that no constructor would produce (no negative balances, no broken size/capacity relationships, etc.).
* If a constructor rejects the parsed values (precondition fails), deserialization fails with a typed `ParseError` (`InvalidType` / a validation variant) — it never yields a half-built or invalid object.
* `data class`es are pure data with a total constructor, so they deserialize directly. Types with **non-trivial invariants** must expose a constructor/factory that validates, or provide a custom `Parser<T>` (§9) that calls it; the compiler will not derive a `Parser` that skips validation.

> Contrast with `deepCopy` (`util.md` §7.7): `deepCopy` duplicates *already-valid in-program data* and so copies fields directly without re-running constructors. Deserialization handles *untrusted external data* and therefore must go through the constructor. The asymmetry is deliberate.

### 11.8 Excluding Fields (`@transient`)

`ref` and `ptr<T>` fields are excluded automatically (§11.1). A **value or `own` field** can be excluded explicitly with `@transient` — the Bestie answer to "don't put this in the payload" (secrets, caches, derived data):

```bestie
data class Session {
    userId: int
    @transient token: str      // never serialized
    @transient cache: Digest    // derived; rebuilt, not stored
}
```

Rules:

* A `@transient` field is **omitted on serialize** and **not read on deserialize**.
* Because deserialization goes through the constructor (§11.7), a `@transient` field must be supplied by the constructor — typically via a default or a derived value. A type whose constructor *requires* a transient field with no default cannot be auto-derived and needs a custom `Parser<T>`.
* Exclusion is **opt-out per field and visible at the declaration site** — there is no way to accidentally serialize a field you marked transient, and no hidden global "skip" list.

### 11.9 Versioning and Schema Evolution

Bestie has **no `serialVersionUID` and no implicit version stamp**. Compatibility is structural and explicit, governed by the §7 mapping rules plus these additions:

* **Strict by default.** Missing required fields and unknown extra fields are errors (§7). This makes incompatibility loud rather than silent.
* **Backward-compatible reads via defaults.** A field with a declared default is **optional on deserialize** — older payloads that lack it parse successfully and the default is applied (through the constructor, §11.7). This is the supported way to add a field without breaking old data.
* **Tolerating unknown fields is opt-in.** A caller may pass an explicit "ignore unknown fields" option to a codec to accept forward-compatible payloads; it is never the default.
* **Renames/removals are breaking** and must be handled by a custom `Parser<T>` (e.g. reading an old field name and mapping it). There is no automatic field aliasing.

This keeps schema evolution explicit and reviewable: a field becomes optional only when it has a default, and lenient parsing is always a visible decision at the call site.

---

## 12. Intentional Non-Features

This library intentionally avoids:

* Runtime reflection
* Schema auto-generation
* Implicit defaults
* Loose typing
* Partial parsing
* A required universal "document as map" representation

Correctness is preferred over convenience.

---

## 13. Summary

`bestie.lib.format` is:

* Uniform
* Predictable
* Explicit
* Safe
* Open to new codecs through the same protocol model

It provides **data interchange**, not data guessing.

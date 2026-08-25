# `bestie.framework.reflection`

Type introspection that is compile-time first and runtime-only when explicitly requested.

## Purpose

`reflection` exposes structured information about types, fields, methods, and annotations so that frameworks (serialization, ORM, DI, validation) can be written generically — **without** paying for runtime reflection by default.

It exists to:

- Make type metadata available to **compile-time** code generation and compiler plugins
- Provide a small, explicit **runtime** reflection surface for the rare cases that genuinely need it
- Keep introspection consistent with Bestie's guarantees: visibility, ownership, immutability, and zero hidden cost

In Bestie, reflection is a tool for **frameworks**, not a substitute for explicit design. Prefer compile-time reflection; reach for runtime reflection only when the set of types is not known until runtime.

## Layering and Dependencies

`reflection` is a `std-framework` module built on the core operators `typeOf(x)` and `sizeOf(T)` (see `core/base.md` §15) and the compile-time annotation model (see `core/annotations.md`).

It is consumed by other `std-framework` modules (`orm`, `di`, `web`, `test`) and by application frameworks.

Import style (explicit per-symbol):

```bestie
import bestie.framework.reflection.TypeInfo
import bestie.framework.reflection.FieldInfo
import bestie.framework.reflection.reflect
import bestie.framework.reflection.Reflectable
import bestie.framework.reflection.Mirror
```

## Design Principles

1. **Compile-time by default** — `reflect<T>()` is resolved at compile time and produces no runtime metadata unless explicitly materialized.
2. **Opt-in at runtime** — runtime metadata exists **only** for types annotated `@Reflectable`. Nothing is retained in the binary otherwise.
3. **No global type scanning** — there is no "find all types" registry. Reflection always starts from a concrete `T` or a `@Reflectable` value.
4. **Respects the type system** — reflection never bypasses visibility, ownership, mutability, or immutability. It cannot read `private` members from outside, nor write a `val` field.
5. **Explicit cost** — the cost of runtime reflection is visible in the binary (materialized metadata) and at the call site (`Mirror.of`).
6. **No magic mutation** — reflection never silently constructs, mutates, or rebinds values.

## Two Tiers of Reflection

| Tier | Entry point | When metadata exists | Cost |
|---|---|---|---|
| Compile-time | `reflect<T>()` | Always available to the compiler/plugins | Zero runtime cost; lowered to specialized code |
| Runtime | `Mirror.of(value)` | Only for `@Reflectable` types | Materialized metadata + lookup at runtime |

Compile-time reflection is the primary mechanism. Runtime reflection is a deliberate, narrow escape hatch.

## Core Types

### `TypeInfo`

Structured description of a type, available in compile-time contexts.

```bestie
class TypeInfo {
    name: str
    size: int
    kind: TypeKind                 // Class, DataClass, ValueClass, Enum, Protocol, Primitive
    fields: list<FieldInfo>
    methods: list<MethodInfo>
    annotations: list<AnnotationInfo>
}
```

### `FieldInfo`

```bestie
class FieldInfo {
    name: str
    typeName: str
    isMutable: bool                // true for `var`, false for `val`
    visibility: Visibility         // Public, Protected, Private
    annotations: list<AnnotationInfo>

    // Compile-time projection: reads this field from a value of the owning type.
    // `FieldType` denotes the field's concrete static type, resolved by the
    // compiler during lowering — there is no boxing and no `any`.
    fun read<T>(value: T): FieldType
}
```

### `MethodInfo`

```bestie
class MethodInfo {
    name: str
    params: list<ParamInfo>
    returnTypeName: str
    isVirtual: bool
    visibility: Visibility
    annotations: list<AnnotationInfo>
}
```

### `AnnotationInfo`

```bestie
class AnnotationInfo {
    name: str
    args: map<str, str>            // compile-time constant arguments
}
```

## Annotation Reference

| Annotation | Target | Effect |
|---|---|---|
| `@Reflectable` | class / data class | Materializes runtime metadata for this type so `Mirror.of` works |
| `@Reflectable(deep = true)` | class | Also materializes metadata for the types of its fields |
| `@Hidden` | field / method | Excludes the member from runtime metadata (still visible to compile-time reflection within visibility rules) |
| `@ReflectName(name)` | field / method | Overrides the reflected name (e.g. for serialization mapping) |

All annotation arguments are compile-time constants, consistent with `core/annotations.md`.

## Compile-Time Reflection

`reflect<T>()` returns a `TypeInfo` usable in compile-time and `const` contexts. The compiler unrolls iteration over `fields`/`methods` into concrete, specialized code — there is no runtime loop over metadata and no boxing.

```bestie
import bestie.framework.reflection.reflect
import bestie.framework.reflection.FieldInfo

// A generic, zero-cost field walker resolved entirely at compile time.
@pure
fun toRecord<T>(value: T): map<str, str> {
    const info = reflect<T>()
    var out = map<str, str>.build()

    // The compiler expands this over the known fields of T.
    for (const field in info.fields) {
        out.put(field.name, field.read(value).toStr())
    }
    return out
}
```

Because `reflect<T>()` is compile-time, `toRecord(user)` compiles to direct field reads — equivalent to hand-written code, with no runtime metadata.

## Runtime Reflection

Runtime reflection is available **only** for `@Reflectable` types. A `Mirror` is an explicit runtime handle over a value's materialized metadata.

```bestie
class Mirror {
    static fun of(value: any): Mirror        // requires the dynamic type to be @Reflectable
    fun typeInfo(): TypeInfo
    fun get(field: str): any                 // honors visibility; returns by value/ownership rules
    fun set(field: str, newValue: any): void // only for `var` fields within visibility; else error
    fun invoke(method: str, args: list<any>): any
}
```

Rules:

- `Mirror.of` fails (returns `!`) if the runtime type was not marked `@Reflectable`.
- `get` cannot read `private`/`protected` members from outside their visibility scope.
- `set` is rejected for `val` fields and for `@immutable` values — immutability is never violated.
- `invoke` respects visibility and dispatch rules; it does not bypass `sealed` or non-`@virtual` constraints.
- Reflective access participates in ownership: reading a non-`Copy` field yields a borrow or an explicit copy, never a silent move.

## Example: Compile-Time Serializer

```bestie
import bestie.framework.reflection.reflect
import bestie.framework.reflection.ReflectName

data class User {
    id: int
    @ReflectName("email_address") email: str
}

@pure
fun toJson<T>(value: T): str {
    const info = reflect<T>()
    val sb = StringBuilder.new()
    sb.append("{")
    for (const field in info.fields) {
        sb.append("\"${field.name}\":")
        sb.append(field.read(value).toJsonValue())
        sb.append(",")
    }
    sb.append("}")
    return sb.toStr()
}
```

`@ReflectName` lets the serializer emit `email_address` while the field stays `email` in code. No runtime metadata is produced — the loop is unrolled at compile time.

## Example: Runtime Plugin Inspection

When the concrete type is unknown until runtime (e.g. a dynamically loaded plugin object), opt in explicitly:

```bestie
import bestie.framework.reflection.Reflectable
import bestie.framework.reflection.Mirror

@Reflectable
class HealthCheckPlugin {
    var enabled: bool = true
    fun run(): str { return "ok" }
}

fun describe(value: any): void {
    val m = Mirror.of(value) catch |err| {
        print("not reflectable")
        return
    }
    val info = m.typeInfo()
    print("type: ${info.name}")
    for (f in info.fields) {
        print("  field ${f.name}: ${f.typeName} (mutable=${f.isMutable})")
    }
}
```

## Visibility, Ownership, and Safety

- Reflection observes the **same visibility** as ordinary code; it grants no special access.
- `val` fields and `@immutable` values are never writable through a `Mirror`.
- Reflective reads and writes obey ownership and borrow rules; no operation can produce a dangling reference or a hidden move.
- Reflection cannot construct a type that is `@noConstruct`, nor allocate one that is `@noNew`.

## Performance and Cost Model

- Compile-time reflection: **zero** runtime cost. Metadata is consumed during compilation and lowered to direct code.
- Runtime reflection: each `@Reflectable` type adds a bounded, inspectable metadata table to the binary. `Mirror` lookups are explicit and never implicit.
- A program that never annotates a type `@Reflectable` and never calls `Mirror.of` carries **no** reflection metadata at all.

## Non-Goals

- No runtime metadata for non-`@Reflectable` types
- No global type registry or "scan all types" capability
- No bypassing of visibility, ownership, immutability, or `sealed`/construction constraints
- No runtime proxy or subclass generation
- No implicit serialization, mapping, or wiring — frameworks must request reflection explicitly

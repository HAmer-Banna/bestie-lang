# Annotations

Annotations in Bestie are **compile-time only constructs**. They exist solely to guide the compiler, tooling, and static analysis, and **introduce zero runtime cost**. No annotation metadata is retained in the generated binary unless explicitly materialized by compiler plugins.

---

## Compile-Time Semantics

* Annotations are evaluated and consumed entirely at **compile time**.
* They do **not** generate runtime reflection data.
* They do **not** incur memory overhead or execution penalties.
* They do **not** bypass ownership, visibility, or concurrency rules.
* Their primary roles include:

  * Validation
  * Code generation
  * Static guarantees
  * Optimization hints

This design aligns annotations with Bestie’s philosophy of *compile-time determinism*.

---

## User-Defined Annotations

Bestie allows **user-defined annotations**, provided they remain strictly compile-time.

Although the Bestie compiler itself is **closed-source and not modifiable**, the language supports **annotation extensions via compiler plugins**. These plugins may:

* Interpret custom annotations
* Enforce additional compile-time rules
* Generate code or metadata
* Integrate with external tooling

This enables rich ecosystems (e.g. frameworks) without compromising compiler stability or runtime performance.
Plugins operate at a compile-time boundary and cannot silently change core runtime semantics.

---

## Annotation Syntax

### Definition

```bestie
annotation ValidateRange(min: int, max: int);
```

### Usage

```bestie
@ValidateRange(min = 0, max = 100)
fun setScore(score: int): void;
```

Notes:

* Annotation parameters are **named and typed**.
* All values must be **compile-time constants**.
* Annotations may be applied multiple times unless restricted by the annotation definition.

---

## Predefined Annotations

Bestie ships with a set of **built-in annotations** understood by the compiler and standard tooling. This list is complete: an annotation the compiler acts on is defined here or it is not a core annotation.

| Annotation | Targets | Effect |
| ---------- | ------- | ------ |
| `@immutable` | type, field, local binding | Deep immutability — see `core/immutability.md` |
| `@virtual` | method | Permits dynamic dispatch and overriding — `oop.md` §2.3 |
| `@override` | method | Enforces override correctness — `oop.md` §6 |
| `@trusted` | expression, local binding | Suppresses a compiler-inserted check the programmer undertakes to uphold: range construction (`lang.md` §6.2), checked narrowing (`types.md` §2.1), dropping pointee `const` (`memory.md` §8.8). Searchable by design |
| `@pure` | function | Side-effect free; callable in `const` initializers (`lang.md` §4.1) |
| `@noInline` | function | Suppresses inlining for stack-trace and profiling clarity |
| `@expose` | any declaration | Exposes the element to external tooling with a stable symbol name |
| `@noNew` | class | Forbids `Type.new(...)` at external call sites — `oop.md` §11.8 |
| `@noInit` | class | Suppresses the generated memberwise initializer — `oop.md` §11.8 |
| `@noConstruct` | class | `@noNew` and `@noInit` combined — `oop.md` §11.8 |
| `@since` | any declaration | Records the layer version a symbol first appeared in — below |
| `@deprecated` | any declaration | Marks a symbol for removal; warns at every use site — below |

The exact semantics of each are enforced at compile time.

**Not core annotations.** `@repr(C)` belongs to `bestie.api.foreign` — matching a C header's declared layout is an FFI contract, not a language mode (`memory.md` §18.7). `@layout` and `@stable` do not exist at any layer: the compiler always packs to the minimum valid representation and there is no opt-out (`lang.md` §6.3). Anything else — `@Initialize`, `@Reflectable`, framework routing and test annotations — comes from a compiler plugin or a higher layer, and is an unknown annotation to a plain core build.

---

## Annotation Targets

Annotations may be applied to:

| Target | Example |
| ------ | ------- |
| Type declarations (`class`, `data class`, `value class`, `enum`, `protocol`) | `@immutable class Config { ... }` |
| Functions and methods, including `init` and accessors | `@pure fun area(r: float64): float64` |
| Fields | `@immutable val cache: Buffer` |
| Function parameters | `fun handle(@Named("primary") db: Database)` |
| Local bindings and expressions | `@trusted val s = (input as Score)` |
| Other annotations (composition, below) | `@immutable annotation ValueObject;` |

An annotation declaration may restrict which of these it accepts; applying it elsewhere is a compile-time error:

```
error: '@virtual' is not applicable to a field — valid targets: method
```

Annotations never appear on statements, blocks, or control-flow keywords. `@trusted` is the one that comes closest, and it attaches to the expression or binding whose check it suppresses — never to a block, because there is no `unsafe { }` in Bestie (`memory.md` §15).

### Annotation-on-Annotation (Composition)

When an annotation is applied to another annotation, it behaves as **annotation composition**:

* The annotated annotation implicitly carries **both its own behavior and its parent’s behavior**.
* This models a form of *inheritance-like reuse* without introducing runtime hierarchies.

Example:

```bestie
@immutable
annotation ValueObject;
```

Any usage of `@ValueObject` now also implies `@immutable`.

---

## Evolution — `@since` and `@deprecated`

Bestie's layering is built on the premise that **std-lib and std-api may retire what did not earn its place**, while core does not (`platform.md` §12). These two annotations are how that premise is expressed in source and enforced by the compiler. Without them the layering would be a convention rather than a mechanism.

They are core annotations because the compiler acts on them and because every layer needs them — not because core expects to use them often.

### `@since`

```bestie
@since("1.4")
public fun rotateLeft(n: int): int { ... }
```

Records the version of the **declaring layer** in which the symbol first appeared. `platform.md` §6 versions each layer separately, so `@since("1.4")` on a `bestie.lib` symbol means std-lib 1.4.

A project declares the versions it targets (`modules-and-packaging.md` §3.2). Using a symbol newer than the declared floor is a compile-time error, not a link failure:

```
error: 'rotateLeft' requires std-lib >= 1.4, but this project declares lib = "1.2"
```

This is what makes the version numbers in `platform.md` §6 a contract rather than a label.

### `@deprecated`

```bestie
@deprecated("use 'encodeAll' — this overload cannot report partial failures")
public fun encode(v: Value): str { ... }

@deprecated(
    reason      = "superseded by the allocator protocol",
    replacement = "BumpAllocator",
    since       = "1.6",
    removedIn   = "2.0"
)
public class Arena { ... }
```

| Parameter | Required | Meaning |
| --------- | -------- | ------- |
| `reason` | yes | Why it is going away. May be given positionally as the first argument |
| `replacement` | no | The symbol to use instead; tooling offers it as a fix |
| `since` | no | Layer version in which deprecation began |
| `removedIn` | no | Layer version in which the symbol will cease to exist |

Behavior:

* Every use site produces a **warning** carrying `reason` and, when present, `replacement`.
* A use is an **error** when the project's declared version for that layer is at or past `removedIn` — deprecation windows are enforced, not merely announced.
* A deprecated declaration may use other deprecated declarations without warning, so a package can keep a retiring API working during its window.
* Deprecating a type deprecates nothing else on its own; a deprecated field or method is marked individually.
* `@deprecated` is **compile-time only**, like every annotation: nothing about it reaches the binary.

```
warning: 'Arena' is deprecated since std-lib 1.6 and will be removed in 2.0
         superseded by the allocator protocol
         replace with: BumpAllocator
```

### What may be deprecated

| Layer | Policy |
| ----- | ------ |
| **std-api** | Freely, with a stated `removedIn` |
| **std-lib** | Conservatively, with a stated `removedIn`, **except** cited symbols |
| **core** | Effectively never. Removing core syntax is a `lang` major version and a language break (`platform.md` §6) |

**Cited symbols cannot be deprecated.** A symbol listed in `core/lang.md` §27 is named by a core rule, so retiring it would change what a keyword or operator means. `@deprecated` on such a symbol is a compile-time error:

```
error: 'Iterator.next' is cited by core/lang.md §27 and cannot be deprecated
```

To retire one, the core citation must be removed first — which is a core change, not a library change. That asymmetry is the whole point: `Arena` is deletable precisely because no core rule mentions it, and `Iterator.next()` is permanent precisely because one does.

---

## Usage in the Standard Framework

Annotations are **extensively used** across Bestie’s standard framework, including but not limited to:

* Web frameworks
* ORM and persistence layers
* Dependency Injection systems
* Validation and schema generation

Because annotations are compile-time only, these frameworks achieve:

* Zero reflection overhead
* Strong static guarantees
* Predictable performance characteristics

When a framework genuinely needs to introspect types, it uses `bestie.framework.reflection`, which is **compile-time first** and only materializes runtime metadata for types explicitly marked `@Reflectable` (see `std-framework/reflection.md`).

---

## Third-Party Annotation Conventions

The plugin system enables third-party libraries to establish their own annotation conventions. One well-known pattern is **default initialization**, analogous to Java's Lombok project.

### `@Initialize` (Plugin Convention)

A plugin may provide `@Initialize` to automatically generate zero or default field values for a class, so that every field without an explicit default receives the natural zero for its type:

| Type | Generated default |
| ---- | ----------------- |
| `int`, `int8`, … | `0` |
| `float32`, `float64` | `0.0` |
| `bool` | `false` |
| `str` | `""` |
| `T ?` | `option.Not_Present` (std-lib name; plugin may emit this) |
| Collection types | empty collection |

```bestie
// Provided by a third-party plugin — not built into the core language
@Initialize
class Config {
    maxConnections: int     // plugin generates: = 0
    timeout: float64        // plugin generates: = 0.0
    host: str               // plugin generates: = ""
    debug: bool             // plugin generates: = false
}
```

Without the plugin active, `@Initialize` is an unknown annotation and the compiler still enforces explicit initialization of every field. This keeps the core strict while letting projects opt in to ergonomic defaults through their toolchain.

---

## Design Rationale

Bestie annotations are designed to be:

* Static rather than reflective
* Explicit rather than magical
* Extensible without compiler modification

This ensures annotations remain a **language-level tool**, not a runtime liability.

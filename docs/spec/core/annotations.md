# Annotations

Annotations in Bestie are **compile-time only constructs**. They exist solely to guide the compiler, tooling, and static analysis, and **introduce zero runtime cost**. No annotation metadata is retained in the generated binary.

---

## Compile-Time Semantics

* Annotations are evaluated and consumed entirely at **compile time**.
* They do **not** generate runtime reflection data.
* They do **not** incur memory overhead or execution penalties.
* They do **not** bypass ownership, visibility, or concurrency rules.
* Their primary roles include:

  * Validation
  * Static guarantees
  * Optimization hints
  * Inert metadata for external tools

This design aligns annotations with Bestie’s philosophy of *compile-time determinism*.

---

## Two Kinds of Annotation

Bestie distinguishes **annotations the compiler acts on** from **annotations that are inert data**.

* **Compiler annotations** are the closed set listed under *Predefined Annotations* below. That list is complete: an annotation the compiler acts on is defined there, or it is not one. It cannot be extended.
* **Declared annotations** may be written by any layer or by user code. The compiler checks their targets and their argument types and then records them for external tools. They have **no compiler behavior whatsoever** — they generate nothing, validate nothing beyond their own arguments, and change no declaration's meaning.

```bestie
annotation ValidateRange(min: int, max: int);

@ValidateRange(min = 0, max = 100)
fun setScore(score: int): void;
```

* Annotation parameters are **named and typed**
* All arguments must be **compile-time constants**
* Applying one to a target its declaration does not permit is a compile-time error
* A declared annotation may not carry a compiler annotation — that would be a way to acquire compiler behavior indirectly

### No compiler plugins

**There is no compiler-plugin mechanism.** A declared annotation cannot be given behavior by supplying code that runs during compilation. This follows from the platform pillars rather than from caution:

* **Compilation speed is a hard constraint** (`platform.md` §1). A plugin is unbounded third-party work inside the build, which no amount of care in the compiler can bound.
* **Core defines the meaning of its own syntax** (`platform.md` §12). A plugin that synthesizes fields or methods decides what a declaration *means* — the delegation that rule exists to forbid.
* **"If something can be resolved at compile time, it must be"** (`lang.md` §2) is a promise about *the compiler*, not about a pipeline whose behavior varies per project.

Frameworks that need generated code use an explicit generator that emits `.bst` source you can read, diff, and step through — not an invisible phase that makes the file on disk differ from the program that runs. The annotation is the generator's input; the generated source is a file in your repository.

In particular there is no Lombok-style field synthesis. `oop.md` §11.10's rule that every field must be explicitly initialized has no opt-out, because an escape hatch from an explicitness rule is just the rule being false.

---

## Predefined Annotations

Bestie ships with a set of **built-in annotations** understood by the compiler and standard tooling. This list is complete: an annotation the compiler acts on is defined here or it is not a core annotation.

| Annotation | Targets | Effect |
| ---------- | ------- | ------ |
| `@trusted` | expression, local binding | Suppresses a compiler-inserted check the programmer undertakes to uphold: range construction (`lang.md` §6.2), checked narrowing (`types.md` §2.1), dropping pointee `const` (`memory.md` §8.8). Searchable by design |
| `@pure` | function | Side-effect free; callable in `const` initializers (`lang.md` §4.1) |
| `@noInline` | function | Suppresses inlining for stack-trace and profiling clarity |
| `@expose` | any declaration | Exposes the element to external tooling with a stable symbol name |
| `@since` | any declaration | Records the layer version a symbol first appeared in — below |
| `@deprecated` | any declaration | Marks a symbol for removal; warns at every use site — below |

The exact semantics of each are enforced at compile time.

**Every annotation in this table is a hint, a suppression, or metadata.** None of them changes what a declaration *is*, how it is laid out, or how a call dispatches. That is the dividing line between an annotation and a keyword in Bestie, and it is why `immutable`, `virtual`, and `override` are **keywords, not annotations**:

* `virtual` adds a type word to the object and turns a direct call into an indirect one — it changes layout *and* dispatch (`oop.md` §2.3, `memory-layout.md` §7).
* `override` is a checked assertion about a hierarchy, not a hint — the compiler rejects it when it is false (`oop.md` §6).
* `immutable` changes which methods a type has (`core/immutability.md` §2.1).

All three are contextual keywords: they carry meaning in exactly one position and remain usable as identifiers elsewhere (`lang.md` §3.1.4). An annotation that changed semantics this way would make the zero-cost, no-runtime-effect promise of this document untrue.

For the same reason there are no `@noNew` / `@noInit` / `@noConstruct` annotations. Restricting construction is a **visibility** question, and visibility already answers it: an `init` declared without `public` is `internal`, so `Type.new(...)` is unavailable outside the module, and `private init` narrows it to the declaring type. Declaring any `init` at all removes the compiler-generated memberwise one (`oop.md` §11.4), so one existing rule covers what three annotations used to.

**Not core annotations.** `@repr(C)` belongs to `bestie.api.foreign` — matching a C header's declared layout is an FFI contract, not a language mode (`memory-layout.md` §13). `@layout` and `@stable` do not exist at any layer: the compiler always packs to the minimum valid representation and there is no opt-out (`lang.md` §6.3). Anything else — `@Reflectable`, framework routing and test annotations — is declared by a higher layer and is an unknown annotation to a plain core build.

---

## Annotation Targets

Annotations may be applied to:

| Target | Example |
| ------ | ------- |
| Type declarations (`class`, `data class`, `value class`, `enum`, `protocol`) | `@deprecated("use Config2") class Config { ... }` |
| Functions and methods, including `init` and accessors | `@pure fun area(r: float64): float64` |
| Fields | `@expose val cache: Buffer` |
| Function parameters | `fun handle(@Named("primary") db: Database)` |
| Local bindings and expressions | `@trusted val s = (input as Score)` |

An annotation declaration may restrict which of these it accepts; applying it elsewhere is a compile-time error:

```
error: '@pure' is not applicable to a field — valid targets: function
```

Annotations never appear on statements, blocks, or control-flow keywords. `@trusted` is the one that comes closest, and it attaches to the expression or binding whose check it suppresses — never to a block, because there is no `unsafe { }` in Bestie (`memory.md` §15).

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

## Design Rationale

Bestie annotations are designed to be:

* Static rather than reflective
* Explicit rather than magical
* Extensible without compiler modification

This ensures annotations remain a **language-level tool**, not a runtime liability.

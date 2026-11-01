# Bestie Metaprogramming — Compile-Time Reflection and Processors

This document defines how code is **extended at compile time**: how a library reads the structure of a user's types, and how it generates new code from annotations. It is the mechanism third-party frameworks are built on — a Spring-style web stack, an ORM, a DI container, a serializer, a test runner, a register-map generator for a driver. None of them needs to ship with the compiler.

There are two tiers, both written in ordinary Bestie:

| Tier | What it does | Typical use |
| ---- | ------------ | ----------- |
| **1. Compile-time reflection** | A generic function reads a type's fields, methods, and annotations at compile time, and the compiler unrolls it into direct code | Serialization, row mapping, validation, equality and hashing, debug printing |
| **2. Processors** | A function runs at compile time on every declaration carrying its annotation and **emits new source** | Route tables, DI containers, builders, test registries, bit-flag types, syscall tables |

Tier 1 is preferred: it produces no source at all. Tier 2 is for code whose *shape* depends on the declaration — a function per route, a type per builder.

Both tiers rely on `vision.md` Rule 2: the convenience comes from the compiler, and the program that runs contains exactly what an expert would have written by hand. Neither leaves metadata in the binary. Bestie has **no runtime reflection** (`platform.md`, invariants).

---

## 1. Principles

1. **Written in Bestie.** A framework author writes Bestie, not compiler internals. There are no native compiler plugins: nothing loads a shared library into the compiler, and nothing a dependency ships can perform I/O during a build.
2. **Additive only.** Metaprogramming may *add* declarations. It never modifies, removes, or reinterprets what the user wrote. A declaration means what its source says; that is how core keeps the meaning of its own syntax (`platform.md` §12).
3. **Pure and deterministic.** Compile-time code is evaluated by the compiler under the same rules as `@pure` constant evaluation (`lang.md` §4.1): no I/O, no clock, no randomness, no global state. The same input always produces the same output.
4. **Bounded.** Every compile-time evaluation runs under a step budget. A processor that loops forever is a compile-time error naming the processor, not a hung build (`platform.md` pillar 1).
5. **Visible.** Generated code is real Bestie source. `bestie expand` prints it, error messages point into it, and the debugger steps through it (§4.6).

---

## 2. Tier 1 — Compile-Time Reflection

### 2.1 `reflect<T>()`

`reflect<T>()` is a core built-in, like `sizeOf(T)` (`lang.md` §15.5). It returns a `const` description of `T`:

```bestie
value class TypeInfo {
    val name:        str
    val kind:        TypeKind              // Class, DataClass, ValueClass, Enum, Protocol, Primitive
    val fields:      array<FieldInfo>      // in declaration order
    val methods:     array<MethodInfo>
    val annotations: array<AnnotationInfo>
}

value class FieldInfo {
    val name:        str
    val mutable:     bool                  // var or val
    val ownership:   Ownership             // Value, Own, Ref
    val optional:    bool                  // the field's type is T ?
    val annotations: array<AnnotationInfo>
}
```

`MethodInfo`, `ParamInfo`, and `AnnotationInfo` follow the same pattern. These types are declared in core's prelude, so `reflect` needs no import. Their names are frozen once published (`platform.md` §12).

Reflection respects visibility: code sees exactly the members it could name directly. A `private` field is not in `reflect<T>().fields` outside its type.

### 2.2 Compile-Time Loops — `for (const x in …)`

A `for` whose loop variable is declared `const` runs at compile time. The compiler **unrolls** it: the body is copied once per element, with the loop variable a different constant in each copy, and each copy is type-checked separately.

```bestie
@pure
fun <T> writeJson(value: T, out: ptr<StringBuilder>) {
    out.val.append("{")
    for (const f in reflect<T>().fields) {
        out.val.append("\"${f.name}\":")
        writeJsonValue(value.[f], out)      // value.[f] — the field f, with its own static type
        out.val.append(",")
    }
    out.val.append("}")
}
```

* `value.[f]` reads the field that the constant `f` describes, with that field's static type. Because the loop is unrolled, `value.[f]` is an `int` in one copy and a `str` in the next. There is no boxing and no `any`.
* `writeJson(user, …)` compiles to a sequence of direct field reads and appends — the code you would write by hand.
* A single member can be named directly: `reflect<T>().field("email")` returns its `FieldInfo`, and a name that does not exist (or is not visible) is a compile-time error.
* A `const` loop may iterate only over a compile-time array: `reflect<T>().fields`, `.methods`, `.annotations`, or a `const` array.
* `when` (`lang.md` §25) works inside the body, so a copy can be erased per field: `when (f.optional) { … } else { … }`.

### 2.3 Reading Annotations

A declared annotation (`annotations.md`) is a compile-time **value**. Its parameters are its fields:

```bestie
annotation Column(name: str)
annotation Transient

data class User {
    @Column(name = "user_id") id: int
    email: str
    @Transient session: str ?
}
```

`f.annotation<Column>()` returns `Column ?`, a constant: present with its arguments if the field carries `@Column`, absent otherwise. `f.has<Transient>()` is the `bool` form.

```bestie
for (const f in reflect<T>().fields) {
    when (f.has<Transient>()) { continue }
    const column = f.annotation<Column>()?.name ?: f.name
    sql.append("${column}, ")
}
```

This is all an ORM's row mapper or a validator needs: the annotations are data, and Tier 1 reads them.

### 2.4 Java Reflection's Problems, and Why Bestie Does Not Have Them

Java reflection made frameworks possible and also caused many of the JVM's worst failures. Every one of them comes from reflection happening **at run time, by name, with permission to break rules**. Bestie's reflection happens at compile time, by typed constant, inside the rules:

| Java reflection problem | Cause | Bestie |
| ----------------------- | ----- | ------ |
| **Slow.** Reflective calls are 10–100× slower and block inlining | Lookups and boxing at run time | Compiled away: `value.[f]` is a direct field access, and an unrolled loop is ordinary code the optimizer sees (§2.2) |
| **Typos fail at run time.** `getMethod("findByEmial")` throws `NoSuchMethodException` in production | Members are named by strings | Members are named by constants from `reflect<T>()`. `reflect<T>().field("emial")` is a **compile-time error** naming the type and the nearest match |
| **Breaks on refactoring.** Renaming a field silently breaks a reflective caller | String names are invisible to the compiler | The compiler sees every use. Renaming a field updates or fails every dependent build |
| **Breaks encapsulation.** `setAccessible(true)` reads private fields and writes `final` ones | Reflection can override access control | Reflection sees only what the calling code could name directly (§2.1). It cannot write a `val` field, read a `private` one, or bypass `immutable` |
| **Bypasses constructors.** Deserialization and `Unsafe` create objects without running their constructor, so invariants never hold | Objects can be allocated reflectively | Reflection cannot construct anything. A value is built only by calling an `init` the caller can see, so range checks and validation always run |
| **Deserialization attacks.** A payload names a class, and the framework instantiates it: remote code execution | Types can be looked up by name at run time | There is no lookup by name and no runtime type registry. A decoder only ever builds the **static** type it was instantiated for, so input cannot choose what gets constructed |
| **Type erasure and unchecked casts.** `List<String>` at run time is a `List` of `Object` | Generics erased; reflection returns `Object` | Generics are monomorphized and each unrolled copy has the field's exact type. There is no `any` and no cast |
| **Slow startup.** Spring scans the classpath and builds proxies at launch | Discovery happens at run time | Discovery happens at compile time (aggregate processors, §3.2). Startup does no scanning |
| **Breaks ahead-of-time compilation.** GraalVM native-image needs hand-written reflection config | The set of reflected members is unknown until run time | Every reflected member is known at compile time, and the binary contains no metadata (§1) |
| **Hidden behaviour.** Proxies and interceptors run code nobody called | Runtime proxies and bytecode weaving | No proxies and no weaving. Generated code is visible with `bestie expand` (§3.6) |
| **Security-manager holes**, `--add-opens` flags, module-system breakage | Reflection needs permission to cross boundaries | Nothing to open: reflection never crosses a boundary the code could not cross itself |

**The rules that guarantee this** (each is part of §2–§3):

* `reflect<T>()` exists only at compile time. There is no runtime form, no `Class` object, no `Class.forName`.
* Members are identified by constant descriptors, never by run-time strings.
* Reflection obeys visibility, `val`, and `immutable` exactly as ordinary code does.
* Reflection only reads declarations and fields; it never allocates, constructs, or frees. Construction always goes through a visible `init`.
* Processors only add declarations, and cannot generate code that breaks any of the above — generated code is checked like code you typed.

### 2.5 Limits

* `reflect<T>()` is valid only where a constant is valid. It cannot be called on a run-time value, and it produces nothing at run time.
* Reflection describes declarations, not function bodies.
* Tier 1 cannot declare anything. A function or type whose *shape* depends on the target — one handler per route, one setter per field — needs Tier 2.

---

## 3. Tier 2 — Processors

### 3.1 Declaring a Processor

A processor is a function marked `processor`, declared in the same package as the annotation it serves:

```bestie
annotation Builder

processor fun builder(target: Target<Builder>, out: ptr<Emitter>): void ! ProcessError {
    val t = target.type
    out.val.line("class ${t.name}Builder {")
    for (f in t.fields) {
        out.val.line("    var ${f.name}: ${f.typeName} ?")
    }
    out.val.line("    fun build(): ${t.name} ! MissingField { … }")
    out.val.line("}")
    out.val.line("fun ${t.name}.builder(): ${t.name}Builder { return ${t.name}Builder.new() }")
}
```

* `processor` is a contextual keyword before `fun`, like `export`.
* `Target<A>` gives the annotated declaration (`target.type`, `target.function`, or `target.field`, depending on what the annotation may target) and the annotation's arguments (`target.annotation: A`).
* `Emitter` collects the generated source, line by line, with core interpolation. `out.val.error(target, "message")` reports a compile-time error at the user's annotation, not inside the processor.
* The body is ordinary Bestie under compile-time rules (§3.4). Ordinary `for` loops over `t.fields` run at compile time here because the whole processor does.

**Importing an annotation activates its processor.** Writing `@Builder` on a class runs `builder` on that class. There is no separate registration, and an annotation without a processor remains what `annotations.md` says: inert data.

### 3.2 Aggregate Processors

Some generated code needs every annotated declaration at once: a route table, a DI container, a test registry. An **aggregate processor** takes a list of targets:

```bestie
annotation Test(name: str = "")

processor fun testRegistry(targets: array<Target<Test>>, out: ptr<Emitter>): void ! ProcessError {
    out.val.line("const BESTIE_TESTS : array<TestCase>[] = {")
    for (t in targets) {
        val fn   = t.function.name
        val name = if (t.annotation.name.isEmpty()) fn else t.annotation.name
        out.val.line("    TestCase.new(name: \"${name}\", run: ${fn}),")
    }
    out.val.line("}")
}
```

An aggregate processor runs **once per module**, over the declarations in that module carrying its annotation. A module that wants to include declarations from another module lists that module explicitly (`modules-and-packaging.md`); there is no program-wide or classpath-wide scan.

### 3.3 What a Processor May Emit

| May emit | May not |
| -------- | ------- |
| New top-level types, functions, constants | Members inside the annotated declaration |
| Extension functions on the target (`fun User.builder()`) | Changes to any existing declaration's fields, signature, or body |
| Types that implement protocols (`class UserCodec impl Serializer<User>`) | Declarations whose names collide with user code — a compile-time error |
| Uses of other annotations | Further `processor` declarations |

**One round.** Processors run on the user's source. They do **not** run on generated source, even if it carries annotations. One round makes expansion linear in the size of the program and rules out processors triggering each other without bound (`platform.md` pillar 1 rejects combinatorially expanding macros).

Why not add members, as Lombok does? Because then `class User` would not show what `User` is. An additive processor keeps every declaration complete as written, and the generated additions are found where Bestie already puts additions: extension functions and separate types. Bestie also has less to generate in the first place: fields are read directly, `val` already makes them read-only, and the memberwise `init` already exists (`oop.md` §11.4), so getters, setters, and all-args constructors are not needed.

### 3.4 Compile-Time Rules

A processor body is evaluated by the compiler, never compiled into the program:

* It may use core and `bestie.lib` (collections, strings, format, regex). It may not use `bestie.api`, so it cannot touch files, sockets, processes, the clock, or the environment.
* It may not use `ptr` arithmetic, `foreign`, `asm`, or `@trusted`.
* Memory comes from an arena owned by the compiler and released after the processor returns.
* It runs under a **step budget**. The default is generous for code generation and fixed by the toolchain. A project may raise it in its manifest (`processor-budget` in `[project]`, `std-tools/package-manager.md` §4.1); a dependency cannot raise it for the project that uses it.
* It sees **declarations**, never function bodies. Editing a function body therefore never re-runs a processor.

### 3.5 Caching and Incremental Builds

Because a processor is pure and sees only declarations, its output depends on exactly three inputs: the processor's own code, the annotation arguments, and the declarations of its targets. The compiler caches each processor's output keyed by a hash of those inputs. In a typical edit — changing a function body — no processor runs at all.

### 3.6 Where Generated Code Lives

Generated source is placed in the module as if it were a file named `<module>/@generated/<processor>/<target>.bst`:

* **`bestie expand <file or declaration>`** prints the generated source for a file or a single declaration.
* Compile errors inside generated code show the generated line and the user annotation that produced it.
* `bestie build` writes the generated files under `build/generated/`, so debuggers and editors open them like any other source.
* Generated code is not committed. It is a deterministic function of committed code, cached like any other build output. (Explicit generators that read external input — a `.proto` file, an OpenAPI document — remain `bestie task` steps whose output is committed, `std-tools/package-manager.md` §4.7.)

---

## 4. Examples

### 4.1 Systems — Hardware Flags

```bestie
annotation Bitflags

@Bitflags
enum Perm as uint8 { Read = 1, Write = 2, Exec = 4 }
```

The processor emits a `value class PermSet` over a `uint8` with `contains`, `with`, `without`, `|`, and `&`, each lowering to a single bitwise instruction. The same technique covers a `@Registers` map for an MMIO block and a `@Syscalls` dispatch table.

### 4.2 Backend — Routing

`bestie.framework.web` (`std-framework/web.md`) uses both tiers. An aggregate processor reads every `@Get`/`@Post` method in the module and emits one dispatch function per route plus a router built from a constant table. Tier 1 decodes each `@Body` parameter. A missing path parameter is reported at the user's annotation, at compile time.

### 4.3 Library — Serialization

`bestie.lib.format` derives `Serializer<T>` and `Parser<T>` with Tier 1 alone (`std-lib/format.md` §11.2). No source is generated, and `json.serialize(user)` compiles to direct field writes.

---

## 5. What Bestie Does Not Have

| Feature | Why not |
| ------- | ------- |
| Native compiler plugins | Unbounded third-party code inside the compiler, with I/O and the power to change meanings |
| Runtime reflection, `any`, type registries | Metadata and boxing in every binary. Everything they are used for is done at compile time here |
| Syntax macros, token rewriting, new keywords | A library could change what source means. Processors take declarations in and give source out; they never touch syntax |
| Processors that modify the annotated declaration | Breaks "a declaration means what it says" (§3.3) |
| Processors on generated code | Makes expansion unbounded (§3.3) |
| I/O at build time | Builds stay reproducible and dependencies cannot read or send anything while you compile |

---

## 6. Summary

* **Tier 1** — `reflect<T>()` and `for (const …)` let generic code read fields, methods, and annotations and compile to direct code.
* **Tier 2** — `processor fun` generates new declarations from annotations: per declaration, or once per module for aggregates.
* Both are Bestie, pure, bounded, cached, and visible. They only add code; they never change what the user wrote.
* This is how frameworks are built on Bestie — by the standard project and by anyone else, with the same power.

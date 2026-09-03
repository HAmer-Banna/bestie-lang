# Bestie Language — Predefined Constants

This document defines the **compiler-provided constants** built into Bestie.

Every value described here is:

* **Compile-time known** — resolved entirely by the compiler, never computed at runtime
* **Zero-cost** — each use is substituted as an immediate operand at its use site (see `core/lang.md` §4), occupying zero bytes of data unless its address is taken
* **Compiler-known, not a library API** — no `import` is required, and none of these are functions you call
* **Cross-compilation-correct** — platform-dependent values resolve to the *target's* value, not the host's

These constants exist to drive **conditional compilation** (`when`, `core/lang.md` §25) and to give exact, target-correct numeric bounds. They are the *only* implicitly available constants in the core language; domain constants (π, e, physical units, …) live in the standard library (`bestie.lib.math` and friends), not here.

---

## 1. The `target` Object

`target` is a compile-time constant object describing the **platform the code is being compiled for**.

| Field           | Type  | Example values                                 |
| --------------- | ----- | ---------------------------------------------- |
| `target.os`     | `str` | `"linux"`, `"windows"`, `"macos"`, `"freebsd"` |
| `target.arch`   | `str` | `"x86_64"`, `"arm64"`, `"riscv64"`             |
| `target.bits`   | `int` | `32`, `64`                                     |
| `target.endian` | `str` | `"little"`, `"big"`                            |

All fields are resolved at compile time to string or integer constants. Because they describe the *target*, cross-compiling for another platform yields that platform's values — `target.bits` inside a 32-bit build is `32` even on a 64-bit host.

### 1.1 Custom Flags

Custom flags are declared in the build configuration (`bestie-project.toml`) and accessed via `target.flag`:

```bestie
when (target.flag("ENABLE_SIMD")) {
    // SIMD path — compiled only when the flag is set
}
```

Flags that are not declared are `false` by default — **no runtime check, no linker symbol, no data**. An undeclared flag simply eliminates its branch.

---

## 2. The `build` Object

`build` is a compile-time constant object describing **how** the program is being built (as opposed to `target`, which describes *where* it will run).

| Field         | Type   | Values                  |
| ------------- | ------ | ----------------------- |
| `build.mode`  | `str`  | `"debug"`, `"release"`  |
| `build.debug` | `bool` | `true` in debug builds  |

`build.mode` is set in `bestie-project.toml` and is the same switch that governs signed-overflow trapping (`core/lang.md` §22.2). It is most useful for guarding debug-only code so it is *eliminated* from release binaries:

```bestie
when (build.debug) {
    logInvariants()      // present only in debug builds — zero cost in release
}
```

---

## 3. Numeric Type Limits

Every numeric primitive exposes its bounds as **compile-time associated constants**. They are exact for the type and require no `import`.

### 3.1 Integers

| Constant  | Meaning                          |
| --------- | -------------------------------- |
| `T.MIN`   | Smallest representable value     |
| `T.MAX`   | Largest representable value      |

Available on every integer type — `int8`, `int16`, `int32`, `int64`, `int`, `uint8`, `uint16`, `uint32`, `uint64`, `uint`, and `byte`.

```bestie
val a = int32.MAX      // 2147483647
val b = uint8.MAX      // 255
val c = int8.MIN       // -128
```

The pointer-sized types `int` / `uint` are **platform-dependent**: `int.MAX` resolves at compile time to the target's width (e.g. `9223372036854775807` on a 64-bit target). Like all predefined constants, this is the *target's* value under cross-compilation.

### 3.2 Floating-Point

Available on `float32` and `float64`:

| Constant       | Meaning                                             |
| -------------- | --------------------------------------------------- |
| `T.MIN`        | Most negative finite value                          |
| `T.MAX`        | Largest finite value                                |
| `T.EPSILON`    | Difference between `1.0` and the next representable value |
| `T.INFINITY`   | Positive infinity                                   |
| `T.NAN`        | A quiet NaN                                         |

```bestie
val hi = float64.MAX
val eps = float64.EPSILON
val inf = float64.INFINITY
```

`T.NAN` is never equal to anything, including itself — test with `.isNaN()` (`core/types.md` §2.4), never `== float64.NAN`.

---

## 4. Boolean Literals

`true` and `false` are keyword literals of type `bool` (`core/types.md` §2.5), not library constants. They are listed here only to note that Bestie has no other implicitly available domain constants in core.

---

## 5. Rules

* Every predefined constant is a **compile-time constant** and may be used anywhere a `const` is valid: `const` initializers, `when` conditions, `array<T>` sizes, `range<T>` bounds, and generic arguments.
* They are **compiler-known**, not importable symbols — no `import`, no namespace, no header.
* **Platform-dependent** constants (`target.*`, `build.*`, `int.MAX`, `int.MIN`, `uint.MAX`) resolve to the concrete *target* value at compile time, making cross-compilation correct by construction.
* Using predefined constants in a `when` condition enables **dead-code elimination**: the unchosen branch is never compiled (`core/lang.md` §25).
* None of these constants carry runtime cost; each is emitted as an immediate operand, not a memory load.

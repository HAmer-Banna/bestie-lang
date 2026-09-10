# bestie.api.cli — Command-Line Interface API

This document defines the **Bestie Standard CLI API (`bestie.api.cli`)**.

`bestie.api.cli` turns a process's `argv` into **typed, validated values**, and generates help text from the same declaration. It does nothing else.

---

## 1. Scope and Non-Goals

### 1.1 What This API Provides

* Declarative argument specification — flags, options, and positionals
* Subcommands
* Typed accessors over parsed results
* Generated `--help` output from the specification
* Typed parse errors that say what was wrong and where

### 1.2 What This API Does *Not* Provide

* **Console I/O.** `print`, `println`, `input` belong to `bestie.api.io` and are not repeated here.
* **Process exit.** `exit(code)` belongs to `bestie.api.os`.
* Interactive prompting, TUI, colors, or progress bars
* Configuration files or environment-variable merging
* Shell completion generation
* A command dispatch framework — parsing produces values; what you do with them is your program

---

## 2. Design Principles

1. **Specification and result are different types.** What arguments *may* appear is not the same thing as what *did*.
2. **Declare once.** Parsing and `--help` come from the same specification, so they cannot disagree.
3. **Values are typed at the accessor**, not stringly-typed at the caller.
4. **Errors are typed and positional** — a parse failure names the offending argument.
5. **No I/O.** This package parses; printing the help it generates is the caller's call to `bestie.api.io`.

---

## 3. Namespacing

```text
bestie.api.cli
```

---

## 4. Class Kinds and Ownership Rationale

| Type | Kind | Why |
| ---- | ---- | --- |
| `ArgKind` | `enum` | A closed set: an argument is a flag, an option, or a positional. |
| `ArgSpec` | `data class` | A declaration, fixed once written. Deeply immutable, structurally comparable, and thread-safe by construction — a spec can be a module-level `val` shared everywhere. |
| `Command` | `data class` | Same: a command tree is a declaration. Its collection fields use the `immutable` variation, which `data class` requires (`core/oop.md` §3.1). |
| `ParsedArgs` | `class` | Owns the parsed values. Identity matters — it is the result of one specific parse of one specific `argv`. |
| `CliError` | `errors` | A closed set of parse failures. |

`ArgSpec` and `Command` being `data class` is what lets a program declare its interface at module level:

```bestie
val VERBOSE = ArgSpec.new(
    name: "verbose", short: 'v', kind: ArgKind.Flag,
    required: false, help: "Print each file as it is processed"
)
```

A module-level `val` requires an immutable type (`core/lang.md` §4.2), and `data class` satisfies that with no further ceremony.

---

## 5. Specification

### 5.1 `ArgKind`

```bestie
enum ArgKind {
    Flag,          // --verbose        present or absent, no value
    Option,        // --out FILE       takes exactly one value
    Positional     // FILE             consumed by position
}
```

### 5.2 `ArgSpec`

```bestie
data class ArgSpec {
    name:     str
    short:    char ?          // absent when there is no single-letter form
    kind:     ArgKind
    required: bool
    help:     str
}
```

`short` is `char ?` rather than a sentinel — an argument that has no short form has no short form, and `core/types.md` §8.3 already spells that.

### 5.3 `Command`

```bestie
data class Command {
    name:        str
    help:        str
    args:        list<ArgSpec>.immutable
    subcommands: list<Command>.immutable
}
```

Both collection fields use the `immutable` variation, which `data class` requires and which `freeze()` produces from a list you build:

```bestie
fun buildSpec(): Command {
    val own args = list<ArgSpec>.new()
    args.add(VERBOSE)
    args.add(OUTPUT)

    val own subs = list<Command>.new()

    return Command.new(
        name: "fmt",
        help: "Format Bestie source files",
        args: move args.freeze(),
        subcommands: move subs.freeze()
    )
}
```

A command with an empty `subcommands` list is a leaf. Subcommand nesting is arbitrary depth; parsing walks it left to right.

---

## 6. Parsing

```bestie
fun parse(argv: slice<str>, spec: ptr<Command>): own ParsedArgs ! CliError
```

`argv` is a `slice<str>` — a borrowed view over the array `main` receives (`core/lang.md` §11.1), so parsing copies nothing. `spec` is a pointer because `Command` is a `data class` that may be large and is never mutated.

Rules:

* Parsing is **total**: every element of `argv` is either consumed or produces an error. Unknown flags are `CliError.UnknownArgument`, never silently ignored.
* `--` ends flag parsing; everything after it is positional, even if it begins with `-`.
* `--name=value` and `--name value` are both accepted for an `Option`.
* Short flags bundle: `-abc` is `-a -b -c` when all three are `Flag`s.
* Parsing performs **no I/O** and never terminates the process — not even for `--help` (§8).

---

## 7. Results

```bestie
class ParsedArgs {
    fun flag(name: str): bool
    fun option(name: str): str ?
    fun optionInt(name: str): int ? ! CliError
    fun positional(index: int): str ?
    fun positionalCount(): int
    fun subcommand(): str ?
    fun free(): void
}
```

Accessors are **typed at the point of retrieval**, so a caller never re-parses a string the parser already validated:

* `flag` is total — an absent flag is `false`, which is what a flag's absence means.
* `option` returns `str ?` — absent when not supplied.
* `optionInt` returns `int ? ! CliError` — the value, absent if the option was not supplied, or `CliError.InvalidValue` if it was supplied and is not an integer. Three outcomes, three-outcome type (`core/types.md` §8.4).
* Asking for a name the specification does not declare is a **programming error**, so it panics rather than returning absent. A typo in an argument name is a bug in your program, not a runtime condition to handle.

### 7.1 A complete program

```bestie
import bestie.api.io.println
import bestie.api.os.exit

fun main(args: array<str>) {
    val spec = buildSpec()

    val own parsed = parse(args[1..], spec.address()) catch |e| {
        println(renderError(e))
        println(renderHelp(spec.address()))
        exit(2)
    }
    defer parsed.free()

    if (parsed.flag("help")) {
        println(renderHelp(spec.address()))
        return
    }

    val verbose = parsed.flag("verbose")
    val out     = parsed.option("out") else { "a.out" }

    for (i in 0..parsed.positionalCount()) {
        val path = parsed.positional(i) else { continue }
        if (verbose) { println("formatting ${path}") }
        format(path, out)
    }
}
```

`args[1..]` skips the program name, which this package never treats specially — `argv[0]` is the caller's to interpret.

---

## 8. Help

```bestie
fun renderHelp(spec: ptr<Command>): str
fun renderError(e: CliError): str
```

Both return a `str`. **Neither prints.** This package does no I/O, so displaying help is an explicit call to `bestie.api.io` — which is what lets a program send help to `stdout` and errors to `stderr`, or capture either in a test.

`--help` is **not** handled automatically. A parser that intercepts a flag and terminates the process is hidden control flow: the program appears to fall through to its own logic and instead exits inside a library call. Declare `help` as an ordinary `Flag` and check it, as §7.1 does.

Help text is generated from the same `Command` the parser uses, so an argument cannot exist without being documented, and documentation cannot drift from behavior.

---

## 9. Error Model

```bestie
errors CliError {
    UnknownArgument,
    MissingRequired,
    MissingValue,        // an Option appeared with no value after it
    InvalidValue,        // a typed accessor could not convert
    UnexpectedPositional,
    UnknownSubcommand
}
```

Rules:

* Every failure is typed; nothing is reported by printing and exiting from inside the parser
* `renderError` turns one into human text at the boundary, where the program decides what to do with it
* No exceptions, no implicit retries, no partial results — a failed parse yields no `ParsedArgs`

---

## 10. Relationship to Other APIs

| Concern | Package |
| ------- | ------- |
| Printing help or errors | `bestie.api.io` |
| Reading interactive input | `bestie.api.io` (`input`) |
| Process exit codes | `bestie.api.os` (`exit`) |
| Environment variables | `bestie.api.os` |
| Reading files named on the command line | `bestie.api.fs` |

This package deliberately re-exports none of them. An earlier shape declared its own `print` / `println` / `readLine`, which meant two packages owned the same names and a program could get different behavior depending on which it imported.

---

## 11. Summary

`bestie.api.cli` is explicit, minimal, and predictable:

* **Specification and result are separate types.** `ArgSpec` and `Command` declare what may appear; `ParsedArgs` holds what did.
* **One declaration drives both parsing and help**, so they cannot disagree.
* **Accessors are typed**, and `optionInt` distinguishes "not supplied" from "supplied and invalid".
* **Nothing prints and nothing exits.** `renderHelp` returns a string; `--help` is a flag you check. Control flow stays in your program.

It provides what CLI tools, services, and scripts need, with no magic and no second console API.

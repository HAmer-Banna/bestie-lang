# std-api.cli — Command-Line Interface API

This document defines the **Bestie Standard CLI API (`std-api.cli`)**.

`std-api.cli` provides explicit tools for:

* Parsing arguments
* Handling commands and subcommands
* Reading user input (interactive)
* Formatting output

---

## 1. Scope and Non-Goals

### 1.1 What This API Provides

* Argument parsing
* Flags and options
* Command registration
* Standard input/output streams (building on core I/O)
* Simple prompt utilities

### 1.2 What This API Does *Not* Provide

* Shell expansion
* Advanced scripting features
* Built-in logging or formatting frameworks
* Async input loops
* Implicit environment handling

---

## 2. Design Principles

1. **Classes for structured commands**
2. **Functions for stateless utilities**
3. **Ownership and lifetimes explicit**
4. **No hidden allocation**
5. **Composable without framework assumptions**

---

## 3. Namespacing

```text
std.api.cli
```

---

## 4. Core Types

### 4.1 `Argument`

```bestie
class Argument {
    name: str
    value: str
    required: bool
}
```

### 4.2 `Command`

```bestie
class Command {
    name: str
    args: list<Argument>
    fun execute(): void
}
```

Rules:

* Commands are classes
* Execution is explicit
* State is local to command instance

---

## 5. Parsing API

```bestie
fun parseArgs(argv: list<str>, cmd: Command): result<Command, CliError>
```

Properties:

* Returns typed errors
* No exceptions
* No implicit environment

---

## 6. Interactive Input

```bestie
fun readLine(prompt: str): str
fun readInt(prompt: str): int | InputError
```

* Ownership of input strings is explicit
* Blocking is expected
* No background threads

---

## 7. Output API

```bestie
fun print(msg: str): void
fun println(msg: str): void
```

* Minimal formatting
* Deterministic
* Direct to stdout
* Supports standard I/O redirection

---

## 8. Error Model

* Typed errors (`CliError`, `InputError`)
* Explicit handling required
* No runtime exceptions

---

## 9. Summary

`std-api.cli` is:

* Explicit
* Minimal
* Predictable
* Safe

It provides **all CLI tools needed for microservices, scripts, and system tooling** without magic.

This document is **finalized**.

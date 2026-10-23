# `bestie` — Project and Package Manager

This document defines **how a Bestie project is created, described, built, given dependencies, and published**: the `bestie` command, the `bestie-project.toml` manifest, the `bestie.lock` lockfile, dependency resolution, and the registry.

It builds on `core/modules-and-packaging.md`, which defines what a source file, a module, a package, and a project *are*. This document defines the tool that works with them.

> **Terminology.** In Bestie a *package* is a namespace inside a module (`modules-and-packaging.md` §5). The thing you version, publish, and depend on is a **project**. The registry hosts projects. This document never says "package" for a distributable unit.

---

## 1. Design Principles

1. **One tool.** `bestie` creates, builds, runs, tests, resolves, and publishes. There is no separate build system, task runner, or package manager to install or configure.
2. **Reproducible by default.** The same manifest, lockfile, toolchain, target, and profile produce a bit-identical artifact on any machine.
3. **No code runs when you fetch or build a dependency.** No install scripts, no build scripts. Everything a dependency needs from the build is declared, not programmed.
4. **Resolution is predictable.** Adding a dependency never silently upgrades an unrelated one.
5. **Source, not binaries.** Dependencies are published and fetched as source and compiled by the consumer (`modules-and-packaging.md` §13.2).
6. **The compiler stays a compiler.** `bestiec` compiles what it is given. It never fetches, resolves, or runs tasks.

---

## 2. Two Executables

| Executable | Role |
| ---------- | ---- |
| `bestiec` | The compiler. Takes sources and module descriptors, produces objects and artifacts. No network, no resolution, no lockfile. Usable directly by other build systems (Bazel, Make, CMake). |
| `bestie` | The front end. Reads the manifest, resolves and fetches dependencies, drives `bestiec`, runs tests and tasks, publishes. |

A program can always be built with `bestiec` alone, given the sources. `bestie` adds nothing to the language — only to the workflow.

---

## 3. Commands

| Command | Does |
| ------- | ---- |
| `bestie new <name>` | Create a project in a new directory — `--lib`, `--staticlib`, `--dynlib`, or an executable by default |
| `bestie init` | Create a manifest in the current directory |
| `bestie add <project>[@version]` | Add a dependency and update the lockfile — `--dev` for a dev-dependency |
| `bestie remove <project>` | Remove a dependency |
| `bestie update [<project>]` | Raise minimum versions to the newest compatible releases, then re-resolve |
| `bestie build [--release]` | Resolve, fetch, compile, link |
| `bestie run [<target>] [-- args]` | Build and run an executable target — the only one, or the one named |
| `bestie check` | Type-check without generating code — the fast editor loop |
| `bestie test` | Build and run tests (`std-framework/test.md`) |
| `bestie fmt` · `bestie lint` · `bestie doc` · `bestie dbg` | Format, analyze, document, debug (`std-tools.md`) |
| `bestie task <name>` | Run a task declared in `[tasks]` (§4.7) |
| `bestie tree` | Print the resolved dependency graph — `--native` adds linked system libraries |
| `bestie why <project>` | Show which path through the graph requires a project and at what version |
| `bestie vendor` | Copy every dependency's source into `vendor/` for offline or audited builds |
| `bestie audit` | Check the lockfile against the security advisory database |
| `bestie publish` | Publish this project to a registry (§7) |
| `bestie yank <version>` | Withdraw a published version from new resolution (§7.3) |
| `bestie clean` | Remove build output |
| `bestie toolchain install <version>` | Install a toolchain; `bestie` selects the one a manifest pins (§4.1) |

Every command that resolves accepts `--locked` (fail if the lockfile would change) and `--offline` (use only the local cache and `vendor/`).

---

## 4. The Manifest — `bestie-project.toml`

A complete example:

```toml
[project]
name        = "acme/order-service"
version     = "1.4.0"
bestie      = "1.2"
description = "Order intake and fulfilment"
license     = "Apache-2.0"
repository  = "https://example.com/acme/order-service"

[[target]]
name    = "order-service"
kind    = "executable"
modules = ["app", "orders"]

[[target]]
name    = "orders-c"
kind    = "staticlib"
modules = ["orders"]

[dependencies]
"acme/money"    = "2.1"
"bestie/web"    = "1.5"
"acme/internal" = { git = "https://example.com/acme/internal.git", rev = "4f1c2a9" }
"acme/fixtures" = { path = "../fixtures" }

[dev-dependencies]
"bestie/test" = "1.2"

[native]
links = [
  { name = "ssl", kind = "dynamic" },
]

[profile.release]
mode  = "release"
lto   = true
strip = true

[tasks]
gen-schema = ["bestie", "run", "schema-gen", "--", "schema/orders.sql"]
```

### 4.1 `[project]`

| Field | Required | Meaning |
| ----- | -------- | ------- |
| `name` | ✅ | `owner/name` — the registry identity (§7.1). Lowercase, `a-z0-9-` |
| `version` | ✅ | Semantic version `MAJOR.MINOR.PATCH` |
| `bestie` | ✅ | Toolchain version. It pins the language, the compiler, **std-lib, and std-api** together |
| `description`, `license`, `repository` | for publishing | Registry metadata; `license` is an SPDX expression |

`std-lib` and `std-api` are **never dependencies**. They ship inside the toolchain, and `bestie = "1.2"` selects them. That is what lets `@since` (`core/annotations.md`) reject a too-new symbol at compile time: the toolchain version is the floor. If the installed toolchain is older than `bestie`, the build fails and names the version to install; it never silently builds with a different one.

### 4.2 `[[target]]`

| Field | Meaning |
| ----- | ------- |
| `name` | Output name |
| `kind` | `executable`, `library`, `staticlib`, or `dynlib` (`modules-and-packaging.md` §13.1) |
| `modules` | The modules this target builds from. Defaults to every module in the project |
| `static` | Linux only: link musl statically for a dependency-free file (`modules-and-packaging.md` §13.3) |
| `triple` | The target triple. Defaults to the host. An OS of `none` makes the target freestanding (`core/freestanding.md` §2) |
| `entry` | Freestanding: the `export(c)` function the program starts at (`freestanding.md` §3.1) |
| `linker-script` | A linker script passed to the platform linker unchanged (`freestanding.md` §6.3) |
| `link-args` | Extra arguments for the platform linker |
| `float` | `"hard"`, `"soft"`, or `"none"` (`freestanding.md` §10.1) |
| `simd-registers` | `false` forbids all use of vector registers (`freestanding.md` §10.2) |
| `red-zone` | x86-64: `false` by default on freestanding targets (`freestanding.md` §10.3) |
| `stack-size` · `stack-check` · `stack-probes` · `stack-frame-warn` | Stack budget and analysis (`freestanding.md` §8) |

A project with no `[[target]]` table is a single `library` target if it has no `main`, and a single `executable` otherwise. A project published to the registry must contain a `library` target — that is what other projects depend on.

### 4.3 `[dependencies]` and `[dev-dependencies]`

A dependency is a project name and a **minimum version**:

| Form | Meaning |
| ---- | ------- |
| `"acme/money" = "2.1"` | Version `2.1.0` or newer, within major version 2 |
| `"acme/money" = "2.1.3"` | `2.1.3` or newer, within major version 2 |
| `{ git = "...", rev = "..." }` | A git repository at an exact commit. A branch or tag name is rejected — it can move |
| `{ path = "..." }` | A project on the local disk. Not allowed in a published project |
| `{ registry = "corp", version = "2.1" }` | From a named registry other than the default (§7.4) |

There are no upper bounds, no ranges, and no wildcards. A requirement says "I need at least this", and resolution (§5) chooses the version. `dev-dependencies` are visible to tests and tasks only, and never to a project that depends on this one.

**std-framework projects** (`bestie/web`, `bestie/test`, …) are ordinary dependencies from the reserved `bestie/` namespace. They are versioned separately from the toolchain because they change faster (`platform.md` §4), and each declares the minimum toolchain it needs.

### 4.4 Modules

Modules are declared by their own descriptors (`modules-and-packaging.md` §4.2), not in the manifest. A dependency's modules become importable once it is listed in `[dependencies]` **and** the importing module lists that module in its `depends` — the manifest supplies the project, the module descriptor states the architectural dependency.

### 4.5 `[native]`

Native libraries a `foreign fun` resolves against (`std-api/foreign.md` §11.1), and C sources to compile alongside the project:

```toml
[native]
links = [
  { name = "z", kind = "static", search = "vendor/zlib/lib" },
]

[[native.compile]]
sources  = ["csrc/fast_crc.c"]
include  = ["csrc/include"]
defines  = { CRC_TABLE = "1" }
```

`sources` may be C (`.c`), assembly (`.s`, `.S` — preprocessed), or C++ (`vision.md` C3). `native.compile` is **declarative**: the toolchain invokes the target's C compiler with exactly these sources, include paths, and defines. There is no hook to run a program instead. This covers the common reason other ecosystems need build scripts — bundling a C library — without letting a dependency execute arbitrary code on your machine.

### 4.6 `[profile.*]`

Profiles select build settings. `debug` and `release` always exist; others may be added.

| Field | Values | Meaning |
| ----- | ------ | ------- |
| `mode` | `debug`, `release`, `safe` | The core build mode — overflow and check behavior (`core/exceptions.md` §4.2, `core/lang.md` §22) |
| `opt` | `none`, `size`, `speed` | Optimization goal |
| `lto` | `true` / `false` | Link-time optimization across all modules |
| `strip` | `true` / `false` | Remove symbols from the artifact |
| `debug-info` | `none`, `lines`, `full` | Debug information emitted |
| `panic-info` | `full`, `location`, `none` | What a panic carries — `none` strips every panic message and file name from the binary (`core/freestanding.md` §3.3) |

Profiles apply to the whole build, dependencies included, so a release build is release all the way down.

### 4.7 `[tasks]`

Project automation — code generation, asset bundling, deployment steps — is declared as named commands and run with `bestie task <name>`:

```toml
[tasks]
gen-schema = ["bestie", "run", "schema-gen", "--", "schema/orders.sql"]
bench      = ["bestie", "run", "--release", "bench"]
```

Rules:

* A task is a command and its arguments, run without a shell, so there is nothing for a shell to reinterpret — the same rule as `bestie.api.os.spawn`.
* **A task never runs implicitly.** `bestie build` does not run tasks, and a dependency's tasks are never run by the project that depends on it. Generated code is committed, or generated by an explicit `bestie task` step in CI.
* A task may depend on others: `{ cmd = [...], after = ["gen-schema"] }`.

This replaces a separate `bestie.build.toml` and the Make-style `bestie build` of earlier drafts: one manifest, one tool.

### 4.8 Workspaces

A repository with several related projects declares a workspace at its root:

```toml
# bestie-workspace.toml
members = ["services/orders", "services/billing", "libs/money"]
```

Members share one `bestie.lock`, one build cache, and one output directory. A member depends on another by `path`, and `bestie publish` rewrites that to the published version. Every command run at the root applies to all members; `-p <name>` selects one.

---

## 5. Dependency Resolution

### 5.1 Minimal Version Selection

Bestie resolves with **minimal version selection** (MVS):

1. Collect every requirement on each project, from the whole graph.
2. For each project, select the **highest of the minimums** anyone asked for — not the newest version in the registry.

```text
app       requires  acme/money ≥ 2.1,  acme/ledger ≥ 1.0
ledger    requires  acme/money ≥ 2.3

selected: acme/money 2.3.0   (highest minimum — even if 2.9.0 exists)
          acme/ledger 1.0.0
```

Why MVS rather than "newest compatible" with a constraint solver:

* **Deterministic without a lockfile.** The result depends only on the manifests, not on what was published since yesterday.
* **Every selected version is one someone tested.** A library that says "≥ 2.3" was built against 2.3; MVS never hands it 2.9 that nobody in the graph has tried.
* **Upgrades are explicit.** `bestie update` raises minimums and shows the diff in the manifest and lockfile. Adding a dependency never moves an unrelated one.
* **No solver, no backtracking, no unsatisfiable puzzles.** Resolution is linear in the size of the graph.

### 5.2 Major Versions

A new major version is a **different project identity** for resolution. `acme/json` 2.x and 3.x may both appear in one graph — one dependency on each — and both are built. They are distinct modules with distinct types: a `json` 2 value is not a `json` 3 value, which is correct, because a major version is a breaking change.

A single project may depend directly on only one major version of a given project.

### 5.3 The Lockfile — `bestie.lock`

```toml
[[project]]
name     = "acme/money"
version  = "2.3.0"
source   = "registry+default"
checksum = "sha256:9b1c…e04f"
```

The lockfile records the selected version, source, and **content checksum** of every project in the graph. It is written by `bestie` and is always committed — for executables and libraries alike.

* Every fetched source is verified against its checksum. A mismatch stops the build: either the network or the registry is serving different bytes than before.
* `--locked` fails if resolution would change the lockfile. CI uses it.
* Because MVS is deterministic, the lockfile is a verification record, not the source of truth for *which* version — the manifests are.

---

## 6. Building

`bestie build` performs, in order:

1. Select the toolchain the manifest pins (§4.1)
2. Resolve the graph (§5) and verify the lockfile
3. Fetch missing sources into the global cache, verifying checksums
4. Compile modules in dependency order, reusing cached outputs keyed by (toolchain, target, profile, source hash)
5. Compile `[[native.compile]]` sources and link the target (`modules-and-packaging.md` §13.3)

Nothing else happens. No step executes code from a dependency.

| Location | Contents |
| -------- | -------- |
| `~/.bestie/sources/` | Fetched dependency sources, content-addressed, read-only |
| `~/.bestie/cache/` | Compiled module outputs, shared across projects on the machine |
| `~/.bestie/toolchains/` | Installed toolchains |
| `out/<profile>/` | This project's artifacts |

Cross-compiling is `bestie build --target <triple>`. Because dependencies are source, every dependency cross-compiles with no extra work.

---

## 7. The Registry

### 7.1 Names

Every published project is `owner/name`. `owner` is a verified user or organization on the registry, and only its members can publish under it. Namespacing prevents squatting on short names and makes the publisher visible in every `import` and every manifest line.

`bestie/` is reserved for std-framework projects published by the language maintainers.

### 7.2 Publishing

`bestie publish`:

1. Requires a clean working tree, a `library` target, `license`, and no `path` dependencies
2. Builds and tests the project as a consumer would — from the packaged source, not the working tree
3. Uploads a source archive, the manifest, and checksums, signed with the publisher's key

A published version is **immutable**. The same name and version always mean the same bytes. There is no overwrite and no delete.

### 7.3 Yanking

`bestie yank 2.3.1` withdraws a version from **new** resolution — a project that does not already lock it will not select it. Existing lockfiles still resolve it, so a yank never breaks a build that was working. Yanking is for a release with a serious bug or vulnerability; `bestie audit` reports locked versions that are yanked or have an advisory.

### 7.4 Other Registries and Vendoring

Organizations can run their own registry and name it in user or project configuration:

```toml
[registries]
corp = "https://registry.corp.example"
```

`bestie vendor` copies every dependency into `vendor/`, and builds with `--offline` use only that directory and the local cache — for air-gapped environments and for teams that review every line they build.

---

## 8. Security Model

Supply-chain attacks in other ecosystems mostly come through three doors. Bestie closes each by construction:

| Door | Elsewhere | Bestie |
| ---- | --------- | ------ |
| Code that runs at install | npm `postinstall`, pip `setup.py` | No install step runs code |
| Code that runs at build | Cargo `build.rs`, Gradle plugins | No build scripts; native compilation is declarative (§4.5) |
| Silent upgrades | Newest-compatible resolution pulls a just-published malicious patch | MVS selects only versions someone in the graph asked for (§5.1) |

Plus: checksums on every fetch (§5.3), immutable signed publications (§7.2), namespaced names (§7.1), and `bestie audit` (§3).

What remains is the code itself: a dependency still runs with your program's privileges once you run your program. `bestie tree` and `bestie vendor` exist so that what you run can be reviewed.

---

## 9. Summary

* **One tool, one manifest.** `bestie` builds, tests, runs tasks, resolves, and publishes from `bestie-project.toml`; `bestiec` stays a pure compiler.
* **The toolchain version pins core, std-lib, and std-api together.** Only std-framework and community projects are dependencies.
* **Minimal version selection** gives reproducible graphs without a solver, and never upgrades anything you did not ask to upgrade.
* **Source distribution, local compiled cache.** No binary Bestie libraries, no ABI breaks; the C ABI is the one stable binary interface.
* **Nothing executes during fetch or build.** Native C code is compiled declaratively, and tasks run only when invoked.
* **A project is the unit you publish** — `owner/name`, immutable once published, yankable but never deleted.

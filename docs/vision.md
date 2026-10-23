# Bestie — Positioning, Principles, and Commitments

This document says **where Bestie competes, how it decides between speed and elegance, and what it commits to**. Every spec document is measured against it. When a design question has no obvious answer, this is the tiebreaker.

---

## 1. Where Bestie Competes

Bestie is a general-purpose language. Its **home territory** is narrower, and design effort goes there first.

Bestie competes with **C++ and Rust**. Its principle is **speed with elegance**: the performance and control of C++, with a language a working engineer can read, write, and hold in their head.

### 1.1 Territory One — Software That Must Be Fast and Close to the Machine

Programs where performance *is* the product and the programmer needs control over memory layout, allocation, and the instructions that run:

* **Browsers** — layout and rendering engines, JavaScript engines, networking stacks
* **Compilers and language tooling** — compilers, linkers, language servers, build systems
* **Engines** — game, physics, rendering, audio, database storage engines, search engines
* **Virtual machines and interpreters** — bytecode VMs, JIT runtimes, emulators
* **Infrastructure** — proxies, load balancers, message brokers, embedded and firmware code

### 1.2 Territory Two — Ending the Two-Language Stack

Many companies run two languages for one product: a high-level language for the application, and C++ for the layer where speed matters.

| Industry | High-level layer | Performance layer |
| -------- | ---------------- | ----------------- |
| Trading and brokerage | Java, C# — order management, risk, client APIs | C++ — market-data handlers, matching, low-latency gateways |
| Maps, traffic, ride-hailing | Java, Python, Go — services, dispatch | C++ — routing engines, map matching, geospatial indexes |
| Ad-tech, telecom | Java, Go — campaign and session logic | C++ — bidders, packet processing |
| Media | Python, Java — pipelines, catalog | C++ — codecs, transcoding |
| ML serving | Python — orchestration | C++ / CUDA — inference kernels |

The two halves are joined by JNI, P/Invoke, pybind, or a network hop, with two build systems, two hiring pipelines, and a serialization boundary in the hot path. **Bestie lets one team write both halves in one language**: backend code with classes, frameworks, and a package manager, and a hot path in the same language with C-level control — no boundary between them.

### 1.3 Not the Target — Yet or At All

Being honest about this keeps effort where it pays:

* **Replacing C in existing operating systems overnight.** Nobody rewrites Linux. Bestie enters that world the way Rust did: through C interoperability — a driver, a module, a userland tool at a time (§3.4).
* **Exploratory data science.** Pandas and notebooks win on interactivity, not speed. Bestie may power the kernels underneath; it does not compete with the notebook.
* **Throwaway scripting.** Bestie can run a single file (`modules-and-packaging.md` §2.2), but shell and Python remain better glue.

---

## 2. Speed and Elegance — How Bestie Decides

Speed and elegance are both principles, and they sometimes pull apart. Bestie does not split the difference case by case. It follows five rules, in order.

### Rule 1 — The default path compiles to what an expert would write by hand

Whatever the shortest, most natural code does must lower to what an experienced C programmer would write for the same task. No allocation they would not make, no indirection they would not add, no check they could not justify.

*In the spec:* operators on user types lower to statically dispatched, inlined calls (`lang.md` §15.3). `array<T>[N]` is a C array (`types.md` §5). A `val` parameter of a large value type is passed by pointer (`memory.md` §6.1). Objects carry no header (`memory-layout.md`). Extension functions resolve statically (`fp.md` §11).

### Rule 2 — Elegance comes from moving work to compile time, not to run time

When a feature makes code nicer, it must do so by having the **compiler** do more: generics, compile-time value parameters, `when`, range-constrained types, `@pure` constant evaluation, monomorphization. A convenience implemented by a runtime mechanism — reflection, boxing, dynamic dispatch by default, a garbage collector — is rejected.

*In the spec:* `fun <const N: int>` gives one source for every array length, each compiled exactly (`lang.md` §7.2). `type Port as uint16 in 1..=65535` checks once at construction and never again (`lang.md` §6.2). `when` erases the losing branch from the binary (`lang.md` §25).

### Rule 3 — A cost that remains has a name at the call site

If a convenient operation must cost something — a copy, an allocation, a lock, a system call — that cost is visible where it is paid: `.toList()`, `move`, `freeze()`, `copyFrom`, `resolve()`, `Lock.acquire()`. Nothing allocates or copies behind syntax that looks free.

*In the spec:* slicing never copies, and the copy has a name (`types.md` §6). DNS is a function, never a hidden step in `connect` (`network.md` §5.3). There is no `toArray()` that would have to invent a length (`collections.md` §9).

### Rule 4 — When both cannot be had, core chooses speed and a higher layer adds the elegance

If a feature cannot be both fast and elegant, **core** takes the fast form, and an opt-in library or framework builds the comfortable form on top. Code that does not want the comfort pays nothing for it.

*In the spec:* core has `array<T>`; `list<T>` lives in `bestie.lib.collections`. Core has raw `ptr<T>`; frameworks build on ownership. `bestie.api.network` has raw sockets; `bestie.api.http` and `bestie.framework.web` are layered on them.

### Rule 5 — The complexity budget is part of performance

A feature that needs whole-program inference, a lifetime solver, or combinatorial template expansion is rejected even if it is fast at run time (`platform.md` §1). Compile time and the reader's attention are performance too. This is the line Bestie draws against both C++ and Rust.

### Checking a Proposal

| Question | A "no" means |
| -------- | ------------ |
| Does the natural spelling compile to what an expert would hand-write? | Redesign, or move it to a library (Rule 1, 4) |
| Is the convenience delivered by the compiler rather than a runtime mechanism? | Reject the runtime mechanism (Rule 2) |
| Is every remaining cost named at the call site? | Add the name, or remove the cost (Rule 3) |
| Can a reader predict what it costs without reading the implementation? | Simplify (Rule 5) |

---

## 3. The Five Commitments

### 3.1 Bare Metal and Low-Level Programming — Fully Supported

Bestie must be usable with **no operating system and no runtime**: firmware, bootloaders, kernels, drivers, hypervisors.

**Already in the spec:**

* A freestanding tier with core and std-lib only — no OS, no hosted I/O (`platform.md` §8)
* No garbage collector, no hidden allocation, no object headers, sized deallocation (`memory-layout.md`)
* `ptr<T>`, pointer arithmetic, casts, alignment queries, `@repr(C)` (`memory.md` §8)
* Volatile, non-elidable MMIO and explicit barriers (`std-api/memory.md`)
* Atomics with explicit ordering that need no OS (`std-lib/concurrency.md` §5)
* Compile-time target selection with `when` and `target.*` (`lang.md` §25)
* Static C-layout arrays in every dimension (`types.md` §5)

**Specified in `core/freestanding.md`** — B1–B8 below are no longer gaps; the table records what each one covers:

| # | Item | Why bare metal needs it |
| - | --- | ----------------------- |
| B1 | **Inline assembly** — an `asm` form with explicit inputs, outputs, and clobbers | Control registers, interrupt enable/disable, system calls, context switches |
| B2 | **Freestanding runtime contract** — a custom entry point, a user-supplied panic handler, and a compile-time error when heap allocation is used with no allocator registered | There is no `main`, no process to terminate, and no heap until you build one |
| B3 | **Linker control** — section placement, fixed addresses, keep/used markers, weak symbols, linker scripts from the manifest | Vector tables, boot sections, memory maps |
| B4 | **Calling conventions** — interrupt handlers and naked functions per architecture | Exception vectors and trampolines |
| B5 | **Untagged unions and bit-fields** for `@repr(C)` types | Hardware registers and C headers use both |
| B6 | **Stack discipline** — per-target stack limits, stack probes, a guarantee of no implicit large stack use | Kernel stacks are small and fixed |
| B7 | **Code-generation hints** — `@inline`, `@noinline`, `@cold`, branch likelihood | Hot interrupt paths and boot code |
| B8 | **Targets without an FPU and without unwinding** — soft-float, and no unwind tables in any mode | Microcontrollers and kernels |

### 3.2 Speed and Elegance in Balance

Section 2 is the commitment. Every new spec document states which rules it relies on, and the performance-critical features below are on the roadmap because both territories need them:

| # | Gap | Who needs it |
| - | --- | ------------ |
| P1 | **SIMD** — portable vector types plus per-target intrinsics behind `when` | Engines, browsers, codecs, trading analytics |
| P2 | **Allocator-aware collections and `new`** — an arena per frame, request, or message | Game engines, trading gateways, compilers |
| P3 | **Fallible element access** — `xs.at(i): T ?` beside panicking `xs[i]`, for indices from untrusted input | Backend services, parsers |
| P4 | **Profile-guided and link-time optimization** in the toolchain | Every hot path |

### 3.3 Extending Easily for Backend Needs

Territory two lives or dies on backend productivity.

**Already in the spec:** streams, files, sockets, HTTP, databases, IPC, processes (`std-api`); web, ORM, DI, templates, testing, streaming (`std-framework`); fibers with I/O parking (`std-lib/concurrency.md`); a package manager with a registry and reproducible builds (`std-tools/package-manager.md`).

**Roadmap:**

| # | Gap | Why |
| - | --- | --- |
| E1 | **Scalable I/O under fibers** — specify the scheduler's poller on io_uring, epoll, kqueue, and IOCP | Hundreds of thousands of connections per process |
| E2 | **Official TLS project** (`bestie/tls`) | Every backend speaks TLS; `http.md` defers it today |
| E3 | **Observability** — structured logging, metrics, tracing in std-framework | Production services need them on day one |
| E4 | **Fault containment for services** — guidance and support for supervisor processes, since a panic ends the process | One bad request must not take down a gateway; `ipc` provides the channel |
| E5 | **Zero-copy serialization** in `bestie.lib.format` | Message-heavy systems (trading, ad-tech) |

### 3.4 Easy Integration with C and C++

Bestie enters the operating-system world, and existing C++ codebases, gradually. That only works if mixing is easy in both directions.

**Already in the spec:** `foreign fun` with C type mapping and explicit ownership, `@repr(C)`, callbacks (`std-api/foreign.md`); `export(c)` with a generated C header; `staticlib` and `dynlib` targets; `[native]` linking; compiling C sources in the same build; run-time `openLibrary` (`modules-and-packaging.md` §13, `foreign.md` §11).

**Roadmap:**

| # | Gap | Why |
| - | --- | --- |
| C1 | **`bestie bind`** — generate `foreign` declarations and `@repr(C)` types from C headers. Run explicitly, output committed, never at build time | Hand-writing bindings for a large C API does not scale |
| C2 | **C variadic calls** (`printf`-style) and C enums, unions, and macro constants in bindings | Common in real C headers |
| C3 | **C++ sources in `native.compile`**, and generated `extern "C"` shims for C++ classes, plus a generated `.hpp` with RAII wrappers for Bestie exports | Incremental adoption inside C++ codebases |
| C4 | **Mixed-language debugging** — DWARF/PDB that debuggers follow across the boundary | Adoption depends on tooling |
| C5 | **Allocator boundary rules** — who frees memory that crosses, stated once | Bestie's sized allocator cannot free `malloc` memory, and vice versa |

### 3.5 Focus on the Home Territory

Every proposal names which territory it serves. A feature that serves neither — or that serves another language's niche at a cost to these two — needs a strong reason. The roadmap above is ordered by what browsers, compilers, engines, VMs, and two-language stacks need first.

---

## 4. Summary

* **Bestie competes with C++ and Rust**, on speed with elegance.
* **Home territory one:** software that must be fast and close to the machine — browsers, compilers, engines, VMs, infrastructure, firmware.
* **Home territory two:** companies that run a high-level language plus C++ can use one language for both.
* **Speed and elegance are balanced by rule, not by taste:** the default path is what an expert would write; elegance comes from the compiler; every remaining cost is named; when both cannot be had, core is fast and a library is elegant; and complexity is a cost too.
* **Five commitments** — bare metal, balance, backend, C/C++ integration, focus — each with what the spec already guarantees and a numbered roadmap of what remains.

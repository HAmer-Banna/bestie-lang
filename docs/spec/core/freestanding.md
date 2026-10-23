# Bestie Language — Freestanding and Low-Level Programming

This document defines what Bestie provides for code that runs **without an operating system, or as part of one**: firmware, bootloaders, kernels, drivers, hypervisors, and the hottest paths of ordinary programs.

It closes the bare-metal commitments B1–B8 of `vision.md` §3.1:

| # | Commitment | Section |
| - | ---------- | ------- |
| B1 | Inline assembly | §4 |
| B2 | Freestanding runtime contract — entry, panic handler, allocator, global state | §3 |
| B3 | Linker control | §6 |
| B4 | Calling conventions — interrupt handlers, naked functions | §5 |
| B5 | Unions and bit-fields | §7 |
| B6 | Stack discipline | §8 |
| B7 | Code-generation hints | §9 |
| B8 | No FPU, no SIMD registers, no unwinding | §10 |

Everything here follows the rest of the language: explicit, compile-time resolved, searchable, and zero-cost when unused. A hosted program that uses none of it pays nothing for its existence.

---

## 1. Principles

1. **No hidden runtime.** In a freestanding build, the only code in the binary is code the program wrote or called. There is no startup routine, no allocator, and no panic printer unless the program supplies one.
2. **Every escape from the safe language is a searchable word.** `asm`, `union`, `global`, `export(interrupt)`, `@trusted` — a reviewer can find every place where the programmer, not the compiler, upholds the rules.
3. **The toolchain, not the source, describes the machine.** Memory maps belong in linker scripts; floating-point and register policy belong in the target. Source code names sections and symbols; it does not hard-code an address map.
4. **Missing capabilities are compile-time errors.** Allocating with no allocator, using `float` on a target with no FPU, or calling `thread.new` with no OS is rejected when the program is built, not discovered when it crashes.

---

## 2. Hosted and Freestanding Targets

Every build is for one of two kinds of target:

| | Hosted | Freestanding |
| - | ------ | ------------ |
| Examples | `x86_64-linux`, `arm64-macos`, `x86_64-windows` | `thumbv7em-none`, `x86_64-none`, `riscv64-none` |
| Entry | `main` (`lang.md` §11), after the runtime initializes args, environment, and stdio | A symbol the target names (§3.1); nothing runs first |
| Panic | Prints and terminates per build mode (`exceptions.md` §4.2) | Calls the program's panic handler (§3.3) |
| Heap | Default size-classed allocator, replaceable (§3.4) | None until the program registers one (§3.4) |
| `thread` | Available | Compile-time error |
| `fiber` | Available | Available once an allocator is registered — fibers need stacks |
| std-lib | All of it | All of it except what needs a heap, unless one is registered |
| std-api | All of it | `bestie.api.memory` only, supplied by the platform or a board-support project |

The kind is a property of the target triple: an OS component of `none` is freestanding. Code tests it at compile time:

```bestie
when (target.freestanding) {
    // bare-metal path
} else {
    // hosted path
}
```

`target.freestanding` and `target.float` (§10) join the `target` object (`constants.md` §1).

A freestanding target is declared in the project manifest (`std-tools/package-manager.md` §4.2):

```toml
[[target]]
name          = "firmware"
kind          = "executable"
triple        = "thumbv7em-none-eabihf"
entry         = "reset"
linker-script = "link/stm32f4.ld"
float         = "hard"
stack-size    = 8192
stack-check   = "strict"
```

---

## 3. The Freestanding Runtime Contract

### 3.1 Entry Point

A freestanding executable has no `main`. Its entry is the function the target's `entry` field names, which must be an `export(c)` function (`std-api/foreign.md` §4.3) returning `never`:

```bestie
export(c) fun reset(): never {
    zeroBss()
    initClocks()
    run()                    // never returns
}
```

**Bestie emits no startup code.** It does not set the stack pointer, zero `.bss`, copy `.data` from flash, enable caches, or set up an FPU. On a microcontroller the hardware loads the stack pointer from the vector table (§11); on other machines a few instructions of `asm` (§4) or a `.S` file in `[[native.compile]]` do it before the entry is called. A board-support project published on the registry is the normal way to share that code — it is ordinary Bestie and assembly, not a privileged runtime.

Until the startup code has run, the program must not read a `global<T>` (§3.5) whose initial value lives in `.data` or `.bss`. That ordering is the programmer's, exactly as in C.

### 3.2 What Needs Nothing

These work from the first instruction of the entry function, with no runtime and no allocator:

* Every primitive, `value class`, `data class`, `enum`, tuple, `array<T>[N]`, `slice<T>`, `range<T>`, `T ?`, and `T ! E`
* `ptr<T>` and everything in `memory.md` §8
* `const` data, compiled into `.rodata`
* `atomic<T>`, `Ordering`, and barriers — `bestie.lib.concurrency` §5 needs no OS
* `MmioRegion<T>` and the rest of `bestie.api.memory`
* Statically dispatched calls, protocols, generics, and `virtual` dispatch — dispatch needs no runtime tables beyond the program's own `.rodata` (`memory-layout.md` §7)

What needs a heap — `class` instances created with `.new()`, the collections of `bestie.lib.collections`, `str` concatenation — needs §3.4.

### 3.3 Panic Handler

On a freestanding target a panic calls the program's **panic handler**. Exactly one function in the program carries `@panicHandler`:

```bestie
@panicHandler
fun onPanic(info: PanicInfo): never {
    disableInterrupts()
    writeFaultRecord(info)
    while (true) { asm("wfi") }      // sleep until reset
}
```

```bestie
value class PanicInfo {
    message: str          // empty when the profile strips messages
    file:    str          // empty when the profile strips locations
    line:    int32
}
```

Rules:

* A freestanding program with no `@panicHandler` is a link-time error. Two are a link-time error.
* The handler returns `never`: it halts, resets the machine, or loops. There is nowhere to return to.
* A panic inside the handler does not recurse — it executes the target's trap instruction.
* A hosted program may also declare a `@panicHandler`, which runs instead of the default printer — for example to write a crash record before the process ends. The process still ends.
* What `PanicInfo` carries is a profile setting, `panic-info = "full" | "location" | "none"` (`package-manager.md` §4.6). `"none"` removes every panic message and file name from the binary — on a 64 KB microcontroller the strings of bounds-check panics are not free.

### 3.4 Global Allocator

Heap allocation goes through the program's **global allocator**: the default size-classed allocator on hosted targets (`compiler/compiler-architecture.md`), and nothing on freestanding targets until the program registers one.

```bestie
import bestie.lib.allocators.GlobalAllocator

@globalAllocator
val HEAP = BumpHeap.new()

value class BumpHeap impl GlobalAllocator {
    fun alloc(size: uint, align: uint): ptr<byte> ? { ... }
    fun free(p: ptr<byte>, size: uint, align: uint): void { ... }
}
```

Rules:

* The registered binding is a module-level `val` whose type implements `GlobalAllocator` (`std-lib/allocators.md`). Its state lives in atomics or `global<T>` storage (§3.5), because the allocator may be called from any thread or interrupt.
* `free` receives the size and alignment the object was allocated with. Deallocation is **sized** everywhere in Bestie (`memory-layout.md` §10), so an allocator never needs a header to find an object's size.
* At most one `@globalAllocator` per program. On a hosted target, one replaces the default — the usual reason being a trading or game process that wants its own allocator for everything.
* **On a freestanding target with no allocator, any heap allocation is a compile-time error**, reported at the allocating call with the chain that reached it:

```
error: heap allocation in a freestanding program with no @globalAllocator
  --> src/net/frame.bst:41:23
   |  val own buf = list<byte>.new()
   = note: reached from reset → run → handleFrame
```

Because Bestie compiles the whole program with monomorphized generics and closed-world dispatch, the compiler can see every allocation the program can make. A kernel that must never allocate in interrupt context can prove it by building its interrupt handlers into a target with no allocator.

### 3.5 Global Mutable State — `global<T>`

Bestie has no mutable global variables (`oop.md` §15). Bare-metal code still needs storage that lives for the whole program and is written after start-up: kernel data structures, DMA buffers, a scheduler's run queue. That storage is `global<T>`:

```bestie
@section(".dma")
val RX_RING: global<array<byte>[2048]> = global<array<byte>[2048]>.zeroed()

val TICKS: atomic<uint64> = atomic<uint64>.new(0)       // no global<T> needed for atomics

fun rxBuffer(): ptr<array<byte>[2048]> {
    return RX_RING.ptr()
}
```

| Member | Meaning |
| ------ | ------- |
| `global<T>.new(value)` | Storage initialized to `value`, placed in `.data` — `value` must be a constant expression |
| `global<T>.zeroed()` | Storage of all-zero bytes, placed in `.bss`; `T` must be a type for which all-zero bytes is a valid value |
| `g.ptr(): ptr<T>` | The only access — a raw, mutable pointer to the storage |

Rules:

* `global<T>` may only be the type of a module-level `val`. It is a compiler-known built-in like `ptr<T>`.
* Its contents are reached **only through `ptr<T>`**, so every read and write of global mutable state is a raw-pointer operation, with raw-pointer rules: synchronization between threads, interrupt handlers, and the main flow is the programmer's — through atomics, through disabling interrupts, or through a `Lock`.
* `global<T>` makes C's global variable available and visible, not easy. Hosted code almost never needs it: module-level `val` atomics and owned objects passed down from `main` cover ordinary programs.

---

## 4. Inline Assembly — `asm`

### 4.1 The `asm` Expression

`asm` embeds target instructions with **explicit operands**. It is an expression: its value is its output operands.

```bestie
when (target.arch == "x86_64") {
    asm("cli")                                                    // no operands

    val cr3 = asm("mov {0}, cr3", out(reg) uint64)                // one output

    val lo, hi = asm("rdtsc", out("eax") uint32, out("edx") uint32)  // two outputs: a tuple

    asm("wrmsr", in("ecx") msr, in("eax") lo, in("edx") hi)       // inputs only

    asm("out dx, al", in("dx") port, in("al") value,
        options(nomem, nostack))
}
```

Operands:

| Operand | Meaning |
| ------- | ------- |
| `in(loc) expr` | `expr` is placed in `loc` before the instructions run |
| `out(loc) Type` | After the instructions, `loc` holds a value of `Type`; it becomes part of the result |
| `inout(loc) expr` | `expr` is placed in `loc`; afterwards the new value of `loc` is part of the result |
| `clobber(loc)` | The instructions overwrite `loc`; the compiler saves anything it needs there |
| `options(...)` | Assumptions the compiler may make — §4.2 |

`loc` is either a specific register — `"eax"`, `"x0"`, `"a0"` — or `reg`, which lets the compiler choose a general-purpose register. Operands with `reg` are referred to in the template as `{0}`, `{1}`, … in operand order. The result of the expression is `void` with no outputs, the output's type with one, and a tuple in operand order with several.

Rules:

* The template is a **string literal** in the target's assembly syntax — Intel syntax on x86, the vendor syntax elsewhere. It is not interpreted by Bestie beyond `{n}` substitution.
* `asm` is meaningful only for one architecture, so it almost always sits inside `when (target.arch == …)`. An `asm` whose template does not assemble for the target is a compile-time error.
* An operand's type must fit its register: integers, `bool`, `ptr<T>`, and on SIMD-capable targets vector types. Anything else is a compile-time error.
* `asm` does not need `@trusted`. Like `ptr<T>`, it is explicit by construction and found by searching for `asm(`. What the instructions do is the programmer's responsibility; the compiler checks only the operand contract.

### 4.2 What the Compiler Assumes

By default the compiler assumes the least: an `asm` expression **may read and write any memory, use the stack, and have side effects**. It is never removed, duplicated, or reordered with other memory operations, so `asm("cli")` and `asm("sti")` really bracket the code between them.

`options(...)` lets the compiler assume more, and so optimize more:

| Option | The programmer promises |
| ------ | ----------------------- |
| `nomem` | The instructions access no memory |
| `readonly` | They read memory but do not write it |
| `pure` | The outputs depend only on the inputs, and there are no other effects — the compiler may remove an unused `asm` or merge two identical ones. Requires `nomem` or `readonly` |
| `nostack` | They do not push to the stack — required on x86-64 targets with a red zone (§10) for any `asm` that pushes |
| `noreturn` | Control never comes back; the expression has type `never` |
| `att` | x86 only: the template uses AT&T syntax |

A false promise is undefined behavior, exactly as a wrong `@trusted` is.

### 4.3 Module-Level Assembly and Assembly Files

Code that is not part of any function — a boot sector, a trampoline, an exception vector written in assembly — is a module-level `asm` declaration, emitted into the object file as written:

```bestie
when (target.arch == "riscv64") {
    asm("""
        .section .text.boot
        .global _start
    _start:
        la   sp, __stack_top
        call kernel_entry
    """)
}
```

Larger assembly lives in `.S` files compiled with the project (`std-tools/package-manager.md` §4.5). Symbols cross in both directions through `export(c)` and `foreign` declarations.

---

## 5. Calling Conventions

### 5.1 ABI Tags on Definitions

`export(c) fun` defines a function with the C calling convention (`std-api/foreign.md` §4.3). The same position takes the other ABI tags a low-level program needs:

| Tag | Use |
| --- | --- |
| `c` | The platform C convention — the default for anything C calls |
| `system` | The OS convention where it differs from C (`stdcall` on 32-bit Windows) |
| `interrupt` | An interrupt or exception handler (§5.2) |
| `naked` | No prologue or epilogue — the body is assembly (§5.3) |
| Architecture-qualified | `x86.sysv`, `x86.win64`, `arm.aapcs`, `arm.aapcs-vfp`, `riscv.lp64d`, … — a named convention when the default is not the one you need |

`foreign(<tag>) fun` declares a function with any of these conventions that is defined elsewhere.

### 5.2 Interrupt Handlers — `export(interrupt)`

An interrupt handler is entered by the hardware, not called. It must save every register it uses and return with the architecture's interrupt-return instruction:

```bestie
import bestie.api.memory.x86.InterruptFrame        // the x86-64 frame layout

when (target.arch == "x86_64") {
    export(interrupt) fun onTimer(frame: ptr<const InterruptFrame>): void {
        TICKS.add(1, Ordering.Relaxed)
        endOfInterrupt()
    }

    export(interrupt) fun onPageFault(frame: ptr<const InterruptFrame>, code: uint64): void {
        recordFault(frame, code, readCr2())
    }
}
```

Rules:

* The permitted signatures are defined per architecture by `bestie.api.memory.<arch>` — on x86-64, a frame pointer with or without an error code; on RISC-V, no parameters; on Cortex-M, where the hardware already saves the caller-saved registers, `interrupt` is the C convention and handlers are ordinary `export(c)` functions.
* An interrupt handler **cannot be called** from Bestie code — that is a compile-time error. It is only referenced by address, to install it: `onTimer.address()` is a `ptr<fn(ptr<const InterruptFrame>) -> void>` for a vector table or an IDT entry.
* A handler on a target with `simd-registers = false` or `float = "none"` cannot touch vector or float registers, so it saves none (§10). That is why kernels use those settings.
* Panicking in a handler calls the panic handler, as anywhere else.

### 5.3 Naked Functions — `export(naked)`

A naked function has no prologue, no epilogue, and no stack frame. Its body is exactly one `asm` expression with `options(noreturn)`:

```bestie
when (target.arch == "x86_64") {
    // void switchContext(from: ptr<Context>, to: ptr<Context>) — SysV: rdi, rsi
    export(naked) fun switchContext(from: ptr<Context>, to: ptr<Context>): void {
        asm("""
            mov [rdi + 0x00], rsp
            mov [rdi + 0x08], rbx
            mov [rdi + 0x10], rbp
            mov rsp, [rsi + 0x00]
            mov rbx, [rsi + 0x08]
            mov rbp, [rsi + 0x10]
            ret
        """, options(noreturn))
    }
}
```

Rules:

* Any statement other than the single `asm` is a compile-time error, because there is no frame for it to use.
* Parameters arrive in the registers of the function's calling convention and are reachable only from the template; naming a parameter in Bestie code inside the body is a compile-time error.
* The template must return or jump itself — hence `noreturn` on the `asm`, while the function keeps its declared signature for callers.

---

## 6. Linker Control

### 6.1 Placement and Retention

| Annotation | Target | Effect |
| ---------- | ------ | ------ |
| `@section("name")` | function, module-level `val` | Emit into the named section instead of the default (`.text`, `.rodata`, `.data`, `.bss`) |
| `@used` | function, module-level `val` | Keep it even when nothing in the program references it — vector tables, boot headers |
| `@weak` | `export(c)` function or `val` | A weak definition: a strong definition of the same symbol elsewhere replaces it |

```bestie
@weak
export(c) fun defaultHandler(): void {
    while (true) { }
}
```

These are linker metadata. They do not change a declaration's type, layout, or dispatch (`annotations.md`).

### 6.2 Data Symbols

`export(c)` applies to module-level `val`s as well as functions, giving data an unmangled symbol name:

```bestie
@section(".vector_table") @used
export(c) val VECTORS: array<ptr<fn() -> void>>[] = {
    reset.address(),
    defaultHandler.address(),
    onSysTick.address(),
}
```

The address of a function or of another module-level `val` is a **link-time constant**, so tables of addresses are valid module-level initializers and land in `.rodata` (or the named section) with relocations — no start-up code fills them in.

Symbols defined elsewhere — by the linker script, by assembly, by C — are declared with `foreign val` (read-only) or `foreign var` (writable):

```bestie
foreign val __stack_top: byte
foreign var __sbss: byte
foreign var __ebss: byte

fun zeroBss(): void {
    var p: ptr<byte> = __sbss.address()
    val end: ptr<byte> = __ebss.address()
    while (p < end) {
        p.val = 0
        p = p.offset(1)
    }
}
```

A linker-script symbol usually has no meaningful value, only an address. Declaring it as `byte` and taking `.address()` is the idiom.

### 6.3 Linker Scripts and Alignment

Bestie does not invent a memory-map language. The memory map — where flash and RAM are, which sections go where, how they are aligned — is a **linker script**, named by the target (`linker-script`, `package-manager.md` §4.2) and passed to the platform linker unchanged.

Alignment of storage follows `memory-layout.md`: there is no `@align`. A section's alignment is set in the linker script; a value that needs more alignment than its type gives it is wrapped in a type that says so:

```bestie
import bestie.lib.allocators.Aligned

@section(".bss.paging")
val PML4: global<Aligned<array<uint64>[512], 4096>> = global<Aligned<array<uint64>[512], 4096>>.zeroed()
```

`Aligned<T, const A: int>` (`std-lib/allocators.md`) is to alignment what `Isolated<T>` is to cache lines: the cost is in the type, never added by default.

---

## 7. Unions and Bit-Fields

### 7.1 `union`

A `union` overlays its fields on the same storage. It exists for C headers and for reinterpreting hardware data; tagged alternatives are an `enum` (`oop.md` §3.3), which is always the right choice in ordinary code.

```bestie
union Word {
    u:     uint32
    f:     float32
    bytes: array<byte>[4]
}

var w = Word.new(f: 1.0)
val bits = @trusted w.u          // 0x3F80_0000 — a reinterpretation
w.u = 0x4000_0000                // writing any field is always allowed
```

Rules:

* Every field is at offset 0. `sizeOf` is the largest field's size rounded to the largest alignment; `alignOf` is the largest alignment. This is C's union layout on every target, so a `union` matches a C header with no further annotation.
* Fields must be copyable value types with no `own` or `ref` content — a union cannot know which field to free.
* `Word.new(...)` initializes **exactly one** field by name. Bytes beyond that field are zero.
* **Reading a field is a reinterpretation** and requires `@trusted`, exactly like a pointer cast (`memory.md` §8.8). The programmer upholds that the bytes are a valid value of the field's type. Writing needs no annotation.
* A `union` is a value type and follows the `val`/`var` rules of `value class`.

### 7.2 `bitfield`

Hardware registers and packed protocol headers name ranges of bits inside one integer. C's bit-fields cannot describe them portably — their bit order and packing are left to each compiler's ABI. Bestie's `bitfield` states every position:

```bestie
enum GpioMode as uint8 {
    Input = 0, Output = 1, Alternate = 2, Analog = 3
}

bitfield Moder : uint32 {
    pin0: GpioMode at 0..=1
    pin1: GpioMode at 2..=3
    pin5: GpioMode at 10..=11
}

bitfield UartCr1 : uint32 {
    enable:     bool  at 0
    rxEnable:   bool  at 2
    txEnable:   bool  at 3
    wordLength: uint8 in 0..=1 at 12
    oversample: bool  at 15
}
```

Rules:

* The backing type is `uint8`, `uint16`, `uint32`, or `uint64`. The bitfield **is** that integer: same size, same alignment, no padding. An `MmioRegion<UartCr1>` reads and writes it with one access of the backing width (`std-api/memory.md` §6).
* Bit `0` is the least significant bit of the backing integer, independent of byte order.
* A position is one bit (`at 3`) or an inclusive range (`at 12..=13`). Ranges must lie inside the backing type and may not overlap — both are compile-time errors.
* A field's type must fit its width: `bool` for one bit; an unsigned integer or range-constrained unsigned type whose range fits; or an `enum as uintN` whose every discriminant fits. When the declared type is wider than the field, the compiler requires a range that fits — `wordLength: uint8 in 0..=1 at 12` — so a value that would not fit cannot reach the field at all (`lang.md` §6.2). Writing a field therefore never needs a run-time check and never silently truncates.
* Bits no field covers are **reserved**. They are zero in `T.new(...)`, preserved by field writes, and visible only through `.bits`.

| Member | Meaning |
| ------ | ------- |
| `T.new(field: value, ...)` | Every field named; reserved bits zero |
| `T.fromBits(raw)` | Reinterpret a raw backing value — all bit patterns are accepted; an `enum` field holding an undeclared value reads as a panic, or under `@trusted` as the raw discriminant |
| `x.bits` | The backing integer |
| `x.field` / `x.field = v` | Shift-and-mask read and write, on a `var` |

```bestie
var cr1 = uart.read(CR1)          // one 32-bit device read
cr1.enable = true
cr1.wordLength = 1
uart.write(CR1, cr1)              // one 32-bit device write
```

Mapping C bit-fields from a header is the job of the binding generator (`vision.md` C1–C2): it computes the target ABI's bit positions and emits a `bitfield` that states them.

---

## 8. Stack Discipline

A kernel stack is a few kilobytes; an interrupt arrives on whatever stack is current; a microcontroller has no guard page. Bestie gives three guarantees and one tool.

**No hidden large frames.** The compiler does not introduce a stack temporary the source did not ask for, other than register spills. A large value passed as a `val` parameter travels by pointer (`memory.md` §6.1); copying a large `array<T>[N]` happens only where the source copies it.

**Frame size warnings.** A function whose frame exceeds the target's `stack-frame-warn` (default 16 KiB hosted, 1 KiB freestanding) is reported with the locals responsible.

**Stack probes.** On hosted targets, every frame larger than a page touches each page in order, so an overflow hits the guard page and faults instead of silently writing past it. Freestanding targets choose with `stack-probes = true | false`.

**Worst-case stack analysis.** With `stack-size` set on a target, the compiler computes the deepest stack any path can reach — from the entry function, and separately from every interrupt handler — and compares it with the budget:

| `stack-check` | Behavior |
| ------------- | -------- |
| `"off"` | No analysis |
| `"warn"` (default) | Over-budget and unbounded paths are warnings |
| `"strict"` | Over-budget and unbounded paths are compile-time errors |

The analysis is exact for most Bestie code, because Bestie compiles closed-world: generics are monomorphized and every `virtual` call has a known set of targets (`memory-layout.md` §7). Two things make a path unbounded: **recursion**, and calls through a **raw function pointer**. Under `"strict"`, the programmer bounds them explicitly with `@stackBound(bytes)` on the function, which the analysis then trusts — a searchable promise like any other. `bestie build --stack-report` prints the deepest path for every root.

---

## 9. Code-Generation Hints

| Hint | Target | Effect |
| ---- | ------ | ------ |
| `@inline` | function | Must be inlined at every call. A call where that is impossible — recursion, a call through a function pointer — is a compile-time error, not a silent fallback |
| `@noInline` | function | Never inlined (`annotations.md`) |
| `@cold` | function | Rarely called: optimized for size, placed away from hot code, and every branch that leads to it is treated as unlikely |
| `@likely` / `@unlikely` | boolean expression | Branch weight for the `if`, `while`, or `else` it controls |

```bestie
@cold
fun reportCorruptPacket(p: ptr<const Packet>): void { ... }

fun handle(p: ptr<const Packet>): void {
    if (@unlikely (p.val.checksum != computeChecksum(p))) {
        reportCorruptPacket(p)
        return
    }
    process(p)
}
```

Hints never change what a program computes. `panic()` and every compiler-inserted bounds check are already `@cold` paths.

**Assumptions.** `assume(cond)` tells the optimizer `cond` holds, so it may drop the checks `cond` makes redundant:

```bestie
fun sumFirst(xs: slice<int>, n: int): int {
    @trusted assume(n <= xs.length)       // bounds checks in the loop fold away
    var total = 0
    for (i in 0..n) { total += xs[i] }
    return total
}
```

`assume` requires `@trusted`. A false assumption is undefined behavior, and the annotation is how a reviewer finds the promise. In `debug` and `safe` builds `assume` is checked and panics when false.

---

## 10. Floating Point, Vector Registers, Red Zone, and Unwinding

### 10.1 Floating Point

The target's `float` setting says what floating point is available:

| `float` | Meaning |
| ------- | ------- |
| `"hard"` | Hardware FPU; floats in float registers |
| `"soft"` | No FPU; `float32` / `float64` arithmetic is compiled to library routines shipped with the toolchain |
| `"none"` | No floating point at all — any `float`, `float32`, or `float64` value in the program is a compile-time error |

`"none"` is what a kernel wants: the kernel does not save the FPU state of the task it interrupted, so it must never use the FPU. `target.float` exposes the setting to `when`, so a library can provide a fixed-point fallback.

### 10.2 Vector Registers

`simd-registers = false` forbids the compiler from using vector registers for anything — including the copies, zeroing, and auto-vectorized loops it would otherwise emit there. Explicit SIMD types (`vision.md` P1) become compile-time errors. Kernels and interrupt handlers use it for the same reason as `float = "none"`.

### 10.3 Red Zone

On x86-64, the System V ABI lets a leaf function use 128 bytes below the stack pointer without moving it. An interrupt arriving on the same stack would overwrite them. Freestanding x86-64 targets therefore default to `red-zone = false`; hosted targets keep the ABI's red zone.

### 10.4 No Unwinding

**Bestie never unwinds.** A panic terminates or calls the panic handler (§3.3); there are no exceptions, no `catch` of a panic, and no destructors run during a panic (`exceptions.md` §4). The compiler therefore emits **no unwind tables** for control flow on any target, in any mode — there is no personality routine, no `.eh_frame` dependency, and no size cost for an unwinding mechanism Bestie does not have.

Frame information for debuggers and crash backtraces is debug information, controlled by `debug-info` and removed by `strip` (`package-manager.md` §4.6). It never affects code generation.

### 10.5 Compiler Support Routines

Some operations need helper routines on some targets: 64-bit division on 32-bit cores, soft-float arithmetic, and `memcpy` / `memset` / `memmove` / `memcmp`, which the compiler emits for large copies. The toolchain ships these, written in Bestie and compiled for the target, and links them only when used. Each is a `@weak` symbol (§6.1): a program or board-support project that provides a faster `memcpy` for its hardware simply defines `export(c) fun memcpy`.

---

## 11. Example — Firmware for a Cortex-M4

A complete program: vector table, reset handler, a GPIO register map as bitfields, a panic handler, and no heap.

```bestie
package firmware

import bestie.api.memory.map
import bestie.api.memory.MemoryRegion
import bestie.api.memory.MmioRegion

foreign val __stack_top: byte
foreign var __sbss: byte
foreign var __ebss: byte

@section(".vector_table") @used
export(c) val VECTORS: array<ptr<fn() -> void>>[] = {
    __stack_top.address().cast<fn() -> void>(),   // word 0: initial stack pointer
    reset.address(),
    defaultHandler.address(),                     // NMI
    defaultHandler.address(),                     // HardFault
}

enum GpioMode as uint8 { Input = 0, Output = 1, Alternate = 2, Analog = 3 }

bitfield Moder : uint32 {
    pin5: GpioMode at 10..=11
}

bitfield Odr : uint32 {
    pin5: bool at 5
}

const GPIOA : uint = 0x4002_0000
const MODER : int = 0                 // register index, in uint32 units
const ODR   : int = 5

export(c) fun reset(): never {
    zeroBss()
    val own moder = map<Moder>(MemoryRegion.new(base: GPIOA, size: 4)) catch |e| { panic("GPIOA") }
    val own odr   = map<Odr>(MemoryRegion.new(base: GPIOA + 0x14, size: 4)) catch |e| { panic("GPIOA") }

    var m = moder.read(0)
    m.pin5 = GpioMode.Output
    moder.write(0, m)

    var on = false
    while (true) {
        on = not on
        odr.write(0, Odr.new(pin5: on))
        delay(1_000_000)
    }
}

fun zeroBss(): void {
    var p: ptr<byte> = __sbss.address()
    val end: ptr<byte> = __ebss.address()
    while (p < end) {
        p.val = 0
        p = p.offset(1)
    }
}

fun delay(cycles: uint32): void {
    for (i in 0..cycles) {
        asm("nop")
    }
}

@weak
export(c) fun defaultHandler(): void {
    while (true) { asm("wfi") }
}

@panicHandler
fun onPanic(info: PanicInfo): never {
    while (true) { asm("bkpt #0") }
}
```

```toml
[[target]]
name          = "blinky"
kind          = "executable"
triple        = "thumbv7em-none-eabihf"
entry         = "reset"
linker-script = "link/stm32f4.ld"
float         = "hard"
stack-size    = 4096
stack-check   = "strict"

[profile.release]
mode       = "release"
opt        = "size"
panic-info = "none"
```

Nothing in the binary was put there by Bestie except the code above and the `memset`-class helpers it used. The program allocates nothing, which the compiler proved, since no `@globalAllocator` exists.

---

## 12. Summary

* **Freestanding means no hidden runtime.** The program supplies its entry, its startup code, its panic handler, and — only if it wants a heap — its allocator. Using a heap without one is a compile-time error.
* **`global<T>` is C's global variable, made explicit:** module-level storage reached only through `ptr<T>`.
* **`asm` is an expression with explicit operands**, conservative by default and optimizable by stated promises; module-level `asm` and `.S` files cover code outside functions.
* **Calling conventions are tags on definitions:** `export(c)`, `export(interrupt)`, `export(naked)`, and named architecture conventions.
* **Linker control is metadata:** `@section`, `@used`, `@weak`, `export(c)` data, `foreign val` symbols, and the platform's own linker scripts. Alignment beyond a type's own is a type, `Aligned<T, A>`.
* **`union` matches C, and `bitfield` states every bit** — reads of a reinterpreted union field are `@trusted`; bitfield writes cannot truncate.
* **The stack is accounted for:** no hidden frames, probes on hosted targets, and a worst-case analysis that is exact for closed-world Bestie code.
* **Hints and assumptions are explicit:** `@inline`, `@cold`, `@likely`, and `@trusted assume`.
* **The target says what the machine allows:** `float = "none"`, `simd-registers = false`, no red zone — and Bestie never needs unwind tables, because it never unwinds.

# Where Bestie Beats C — Code Generation and Memory

This document lists the ways the Bestie compiler can produce **faster code and smaller data than a C compiler**, and the language rule that makes each one legal. It is the companion to `compiler-architecture.md` §5 (how the optimizer is organised) and `spec/core/memory-layout.md` (what layout the language guarantees).

It answers a fair question: decades after C and C++, why would a new language be faster? Not by out-optimizing an expert writing careful C — nothing does that reliably. Bestie wins because its **default code is already the code that expert would write**, and because its rules give the compiler facts a C compiler is never allowed to assume.

---

## 1. The Principle

A C compiler must assume the worst: any two pointers may overlap, any function may be called from anywhere with the C calling convention, and the programmer may depend on the declared order of every field. Each of those assumptions blocks an optimization.

Bestie's rules remove those assumptions for every function and type that does not cross a foreign boundary:

| C must assume | Bestie knows, because | Spec |
| ------------- | --------------------- | ---- |
| Two pointers may alias | `own` is unique, `val` parameters cannot be written, `const`/`immutable` data never changes | `spec/core/memory.md` §5–§6, `spec/core/immutability.md` |
| Any function may be called by foreign code | Only `export` functions are visible outside the program; every Bestie program is closed at link time | `spec/std-api/foreign.md` §4.3, `spec/core/memory-layout.md` §7.1 |
| Field order is the declared order | Layout is the compiler's choice, except under `@repr(C)` | `spec/core/memory-layout.md` §4, §13 |
| Any bit pattern may be a valid value | Ranges, enums, `bool`, `char`, and non-null references have unused patterns | `spec/core/memory-layout.md` §11 |
| A call may throw and unwind | There are no exceptions and no unwinding | `spec/core/exceptions.md`, `spec/core/freestanding.md` §10.4 |

**`foreign` code is the line.** Everything below applies to code the compiler can see in full. A `foreign` function, an `export` function, a `@repr(C)` type, and memory reached through `ptr<T>` follow C's rules, because C code may be on the other side.

---

## 2. Faster Code

### 2.1 No-Alias Facts Everywhere

Rust's biggest code-generation advantage over C is that the compiler knows most references do not overlap. Bestie gets the same facts from its own rules:

* An `own T` binding or field is the only path to its object.
* A `val` parameter cannot be written by the callee, and a large one is passed by pointer only because nothing can observe the difference (`memory.md` §6.1).
* `const` and `immutable` data is never written after construction.

The compiler emits these as `noalias` / `readonly` facts in its IR. In C they correspond to `restrict` and `const` — but C's `const` promises nothing (it can be cast away), and `restrict` is rarely written and never checked. With them, loads can be kept in registers across stores, loops vectorize without runtime overlap checks, and redundant reloads disappear.

`ptr<T>` carries no aliasing facts: raw pointers are the programmer's domain, as in C.

**Status:** compiler obligation for 1.0.

### 2.2 A Private Calling Convention

C's calling convention is fixed by the platform ABI: small structs go to memory, a second return value needs an out-parameter, and every function keeps a conventional entry point.

A Bestie function that is not `export`, not `foreign`, and not called through a `foreign fn` is invisible to C. The compiler is free to choose how it is called:

* **Multiple results in registers.** `T ! E` returns the value and the error tag in two registers (`exceptions.md` §3.6). `T ?` returns the value plus a flag, or nothing extra when `T` has a niche. Small structs and tuples return in registers, never through hidden memory.
* **Unused parameters removed**, and constant arguments folded into a specialized copy when every caller passes the same constant.
* **Callee-saved registers chosen per function** from what it actually uses.
* When a function's address is taken as a `fn`, the compiler emits one conventional-ABI thunk for that use only. Direct calls keep the private convention.

Because the program is closed at link time (`compiler-architecture.md` §5.2), every call site is known, so this is a whole-program decision with no risk of an unknown caller.

**Status:** compiler obligation for 1.0 for multiple returns and `T ! E` / `T ?`; parameter removal and specialization are optimizations the compiler may add later. Neither changes program semantics.

### 2.3 Devirtualization by Construction

C++ virtual calls stay indirect unless the optimizer proves the dynamic type. In Bestie:

* Dispatch is static by default; `virtual` is opt-in (`spec/core/oop.md` §2.3).
* The whole hierarchy is known at link time. A `virtual` method with one implementation becomes a direct call; a small set of implementations becomes a compare-and-branch or a jump table.
* A sealed hierarchy switches on a compact tag with a jump table, and the compiler checks the cases are exhaustive (`memory-layout.md` §8).

**Status:** compiler obligation.

### 2.4 No Unwinding

C++ pays for exceptions even when none is thrown: unwind tables in the binary, and lost optimizations around every call that might throw. Bestie has no exceptions. Errors are values in registers, panics never unwind, and no unwind tables are emitted in any build mode (`freestanding.md` §10.4). Binaries are smaller, and the optimizer may move code across calls freely.

**Status:** language guarantee.

### 2.5 Work Done at Compile Time

Several features that cost run time in other languages cost none in Bestie:

| Feature | Elsewhere | Bestie |
| ------- | --------- | ------ |
| Serialization, row mapping | Runtime reflection or generated tables | Compile-time loops unrolled into direct field access (`spec/core/metaprogramming.md` §2) |
| Regular expressions | Compiled at startup, cached | A `const` regex is a finished DFA in `.rodata` (`spec/std-lib/regex.md` §2.1) |
| Range checks on newtypes | Checked on every use | Checked once at construction (`spec/core/lang.md` §6.2) |
| Platform selection | Runtime flags or preprocessor | `when` erases the losing branch (`spec/core/lang.md` §25) |
| Generic code | Boxing or dynamic dispatch | Monomorphized, with array lengths as constants (`spec/core/lang.md` §7.2) |

**Status:** language guarantees.

### 2.6 Bounds Checks That Disappear

Every index is checked unless the compiler proves it in range. Bestie gives it more to prove with than C has: array lengths are in the type, `for (i in 0..xs.length)` has a known range, and range-constrained integers carry their bounds. A constant index is checked at compile time and emits nothing.

**Proposal — `@checkFree`.** A function annotation, borrowed from Goose, that makes it a compile-time error for any bounds check to remain in the function after optimization. It turns "I hope the checks were removed" into a guarantee on hot paths, and the error names the index the compiler could not prove. It would join the closed list in `spec/core/annotations.md`.

**Status:** check elimination is a compiler obligation; `@checkFree` is proposed (`vision.md` §3.2, P7).

### 2.7 Stack Allocation of Non-Escaping Objects

When an object is created with `new` and freed in the same function, and no pointer to it escapes, the compiler may place it on the stack and remove both the allocation and the free. This only ever removes a cost, so it is consistent with `vision.md` Rule 3: the cost written in the source is an upper bound.

**Status:** optimization the compiler may perform.

---

## 3. Less Memory

### 3.1 Already Guaranteed

These are in `spec/core/memory-layout.md` and are compiler obligations today:

| Technique | Saves | Spec |
| --------- | ----- | ---- |
| Field reordering to remove padding | Typically 10–30% of struct size versus declaration order | §4 |
| No object header; a 32-bit type word only for `virtual` hierarchies | 8–16 bytes per object versus Java, C#, or C++ with a vtable | §2, §7 |
| No allocator header — sized deallocation | 8–16 bytes per heap allocation versus `malloc` | §10 |
| Niches for `T ?` and enum tags | The tag byte, and its padding, on every optional reference, `bool`, `enum`, or range-constrained field | §11 |
| Narrow storage for range-constrained integers | `int in 0..=200` is 1 byte, not 8 | §4.3 |
| Inline value types and arrays | No pointer and no separate allocation per embedded value | §5 |

### 3.2 Tagged Pointers — Tags in Alignment Bits

A heap object is aligned to at least 8 bytes on 64-bit targets, so the low 3 bits of every `own T`, `ref T`, and class reference are always zero. Those bits are niches (`memory-layout.md` §11.1). An `enum` whose variants carry references can store its tag there:

```bestie
enum Expr {
    Num(own NumNode),
    Add(own AddNode),
    Mul(own MulNode),
    Var(own VarNode),
}
```

`Expr` is **one word**, not a word plus a tag padded to two. Compilers, interpreters, and every tree-shaped data structure benefit: an AST, a JSON document, a scene graph. Reading the reference masks the low bits, one `and` instruction that the CPU overlaps with the load.

The niche choice is invisible to programs (`memory-layout.md` §11.3), so this changes no semantics. It never applies to `ptr<T>`, whose low bits the programmer may use, or to `@repr(C)` types.

**Status:** guaranteed niche source in `memory-layout.md` §11.1; the compiler uses it for enums of references.

### 3.3 Compressed References

A 64-bit pointer is often twice what is needed. Java (compressed oops), V8 (pointer compression), and Goose (relative references) use 32-bit offsets inside a bounded region instead, which halves the size of pointer-heavy structures and makes them position-independent — they can be written to a file or placed in shared memory and used as they are.

**Proposal.** A reference type relative to an arena or region, `rel<T>` with a 32-bit (or narrower) offset, checked against the region's size. It would serve the arena-per-request pattern (`vision.md` P2), shared memory (`spec/std-api/ipc.md`), and zero-copy serialization (`vision.md` E5). Because it is a distinct type, its cost and its region are visible at the declaration (`vision.md` Rule 3).

**Status:** proposed (`vision.md` §3.2, P8). Needs a spec of its own.

### 3.4 Struct-of-Arrays Collections

A `list<Particle>` stores whole particles one after another. A loop that reads only `position` still pulls every other field into the cache. A struct-of-arrays layout stores each field in its own array, so that loop reads only positions.

**Proposal.** A `list<T>.soa` representation variation (`spec/std-lib/collections.md` §3) that stores each field of `T` separately. It is a variation, so it changes cost and not API: `ps[i].position` still works. It is built with compile-time reflection (`metaprogramming.md` §2), so it is library code, not compiler magic.

**Status:** proposed (`vision.md` §3.2, P9).

### 3.5 Hot/Cold Field Splitting

With profile data (`vision.md` P4), the compiler can place a class's rarely used fields in a separate block, so the frequently used ones share cache lines. This is a layout decision the compiler already owns (`memory-layout.md` §4), applied with better information.

**Status:** optimization the compiler may perform with PGO.

---

## 4. What Not to Expect

* **Bestie will not beat expert, hand-tuned C on that C's own terms.** The gain is that ordinary Bestie code gets the optimizations an expert would apply by hand, without writing them.
* **Benchmarks must be honest.** When the project publishes performance numbers, it compares against both idiomatic and hand-optimized C and C++, and says which is which.
* **Foreign code keeps C's costs.** At every `foreign` boundary the C ABI, C layout, and C aliasing rules apply in full.

---

## 5. Summary

| Advantage | Kind | Status |
| --------- | ---- | ------ |
| No-alias facts from `own`, `val`, `const` | Speed | 1.0 obligation |
| Private calling convention for non-`export` functions | Speed | 1.0 obligation (results in registers) |
| Devirtualization in a closed world | Speed | Obligation |
| No unwind tables | Speed, size | Guarantee |
| Compile-time reflection, `const` regex, `when` | Speed | Guarantee |
| Bounds-check elimination; `@checkFree` | Speed | Obligation; proposal |
| Stack allocation of non-escaping objects | Speed, memory | Optional optimization |
| Field reordering, no headers, sized free, niches | Memory | Guarantee |
| Tagged pointers in alignment bits | Memory | Guaranteed niche source |
| Compressed `rel<T>` references | Memory | Proposal |
| `list<T>.soa` | Memory, speed | Proposal |
| Hot/cold splitting with PGO | Memory, speed | Optional optimization |

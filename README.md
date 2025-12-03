Bestie Programming Language
“Fast as light. Native with no GC. Control of C. Beauty of Kotlin.”
Elegance with Excellence
🌊 Vision

Bestie is a modern compiled language designed to unify two historically separate worlds:

Systems programming (C, Rust, Zig)

Backend & application development (Java, Kotlin, Go)

Bestie’s mission is clear:

Bring the power, speed, and control of systems languages to backend developers —
while giving system programmers the ergonomics, expressiveness, and elegance of high-level languages.

🔥 Core Principles
No GC — ever

Memory is explicit by design. No garbage collector, no hidden costs.

Native performance

Compiles directly to machine code with deterministic behavior and predictable performance.

Elegant syntax

Inspired by Kotlin: readable, expressive, joyful to write.

Safe control

Bestie introduces a fresh memory model that provides:

C-level control

Rust-like safety

Kotlin-like developer experience

All enforced at compile time

🌱 Language Elements

Bestie includes a minimal yet powerful set of constructs:

package

data class (immutable, inlined)

value class (inlined)

abstract class

single class (inline singleton)

class (final, inlined)

open class

enum

enum class

protocol

group protocol

fun

inline fun

Lambdas

Every class automatically gets:

.new()

.free()

.address()

.deref()

⚡ Memory Model Overview

Bestie exposes memory management in a safe and friendly way:

Allocation
val user = User.new()

Deallocation
user.free()

Address
val ptr = user.address()

Dereference
val u2 = ptr.deref()


No nulls. No unsafe blocks. All validated by the compiler.

🔁 Concurrency Model

Bestie aims for:

Lightweight threads

Structured concurrency

Zero shared-mutable-data by default

Compile-time detection of:

data races

deadlocks

livelocks

improper locks

High-level async built on safe primitives

🧱 Project Structure
/bestie
    /compiler
    /stdlib
    /spec
    /vm (optional)
    /docs
    README.md

🌊 Branching Model

The first branch is:

first-wave


Future development branches may include:

second-wave

memory-core

syntax-layer

type-system

concurrency-engine

🌐 Logo & Branding

Place logos and mascots under:

/branding
    /logo
    /mascot (Finn the Dolphin)


Future automated image generation & SVG refinement to be added later.

🚀 Roadmap (High-Level)

 Define core syntax grammar

 Implement tokenizer

 Implement parser

 Define compiled IR

 Implement allocator system

 Implement class model (data, value, single, class, open class)

 Implement protocol system

 Implement concurrency primitives

 Build standard library v0.1

 Write extensive compiler tests

 Publish Bestie 0.1 preview

🤝 Contributing

Bestie is a language for developers who want both power and elegance.
Contributions are welcome once the core specification stabilizes.
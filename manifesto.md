# The Bestie Manifesto

Bestie is a practical programming language.

Not academic.
Not experimental.
Not clever for its own sake.

Practical by design.

---

## What Bestie Is

Bestie is a native, ahead-of-time compiled language built for:

* Systems programming
* Backend and distributed services
* Bare-metal and embedded software
* Performance-critical applications

With:

* One language
* One mental model
* One deterministic toolchain

No split between “low-level” and “high-level” modes.

---

## What Bestie Believes

### Performance Is a First-Class Feature

Performance is not an optimization phase.
It is a design constraint.

Bestie compiles directly to native code.
There is no garbage collector.
There are no hidden runtime costs.

If performance is optional, it will eventually be ignored.

---

### Safety Means Predictability, Not Restriction

Bestie does not forbid power.
It eliminates **surprises**.

Bestie prevents:

* Accidental ownership/lifetime errors in ownership-validated code paths
* Ambiguous behavior
* Hidden undefined behavior

There is:

* No null
* No silent failure
* No hidden unsafe boundary

Unsafe operations remain explicit at `ptr`, FFI, and manual `free` boundaries.

Safety exists to make intent explicit — not to restrict capability.

---

### Memory Is a Language Responsibility

Programmers control *when* memory is allocated and released.

They should not be forced to manage:

* Padding
* Layout
* Hidden metadata
* Implicit ownership

Bestie guarantees:

* Deterministic memory behavior
* Compile-time validated lifetimes
* Predictable object layout
* Explicit ownership semantics

Memory is something you reason about — not something you hope works.

---

### Abstractions Must Be Zero-Cost

Abstractions exist to improve clarity.

They must **not** impose runtime penalties.

If an abstraction cannot be compiled away,
it does not belong in the core language.

Zero-cost is not an optimization.
It is a requirement.

---

### Paradigms Are Tools, Not Identities

Object-oriented programming is useful.
Functional programming is useful.
Procedural programming is useful.

Bestie supports all of them.
Bestie enforces none of them.

Dogma does not scale.

---

### Determinism Over Magic

Bestie prefers:

* Compile-time resolution over runtime discovery
* Explicit control over hidden automation
* Predictable execution over convenience

There is no hidden runtime behavior.
What you write is what executes.

---

## What Bestie Is Not

Bestie is:

* Not a scripting language
* Not a research experiment
* Not runtime-driven
* Not macro-driven
* Not framework-dependent
* Not trying to solve every problem

There is no hidden magic.
Everything that matters is visible and predictable.

---

## Why Bestie Exists

Because:

* C gives control but limited safety
* C++ gives power but increasing complexity
* Java gives safety but removes control
* Rust gives safety with high cognitive overhead

Modern systems demand:

* Native performance
* Predictable behavior
* Explicit control
* Reasonable complexity

Bestie exists to meet those demands **without compromise**.

---

## The Promise

Bestie promises:

1. Native performance
2. Deterministic and explicit memory control
3. No hidden undefined behavior; unsafe boundaries are explicit
4. Strong compile-time guarantees
5. Long-term language stability

Nothing hidden.
Nothing implicit.
Nothing unnecessary.

---

## Final Word

Bestie is not trying to implement every idea in programming.

It is focused on getting the **fundamentals exactly right**.

Once the fundamentals are correct,
everything else becomes possible.

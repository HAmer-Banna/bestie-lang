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

Bestie compiles to native code.
There is no garbage collector.
There are no hidden runtime costs.

If performance is optional, it will eventually be ignored.

---

### Safety Means Predictability, Not Restriction

Bestie does not prevent programmers from making choices.

It prevents:

* Accidents
* Ambiguity
* Undefined behavior

There is:

* No null
* No silent failure
* No undefined execution

Safety exists to make intent explicit, not to forbid power.

---

### Memory Is a Language Responsibility

Programmers must control *when* memory is allocated and released.

They should not be forced to manage:

* Padding
* Layout
* Hidden metadata

Bestie guarantees:

* Deterministic memory behavior
* Optimal layout by default
* Compile-time validated lifetimes

Memory is something you reason about, not something you hope works.

---

### Abstractions Must Be Zero-Cost

Abstractions exist to improve clarity.
They must not impose runtime penalties.

If an abstraction cannot be compiled away,
it does not belong in the core language.

---

### Paradigms Are Tools, Not Identities

Object-oriented programming is useful.
Functional programming is useful.
Procedural programming is useful.

Bestie supports all of them.

It enforces none of them.

Dogma does not scale.

---

## What Bestie Is Not

Bestie is:

* Not a scripting language
* Not a minimal or “tiny” language
* Not a research experiment
* Not framework-driven
* Not runtime-centric

There is no hidden magic.
Everything that matters is visible.

---

## Why Bestie Exists

Because:

* C gives control but no safety
* C++ gives power but no predictability
* Java gives safety but removes control
* Rust gives safety with high cognitive cost

Modern systems demand:

* Native performance
* Predictable behavior
* Explicit control
* Reasonable complexity

Bestie exists to meet those demands without compromise.

---

## The Promise

Bestie promises:

1. Native performance
2. Explicit and deterministic memory control
3. Zero undefined behavior
4. Professional-grade ergonomics
5. Long-term stability

Nothing hidden.
Nothing implied.
Nothing extra.

---

## Final Word

Bestie is not trying to cover every idea in programming.

It is trying to get the fundamentals exactly right.

Once the fundamentals are right,
everything else becomes possible.

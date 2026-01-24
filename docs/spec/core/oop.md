Bestie Language — Object-Oriented Programming (OOP)

This document defines Object-Oriented Programming in Bestie.

OOP in Bestie is:
	•	Explicit
	•	Compile-time driven
	•	Static by default
	•	Dynamic only when explicitly requested
	•	Memory-aware
	•	Concurrency-safe by construction

Bestie treats OOP as a controlled toolset, not a dominant paradigm.

⸻

1. Core OOP Philosophy

Bestie explicitly rejects:
	•	Implicit dynamic dispatch
	•	“Everything is an object”
	•	Inheritance-heavy hierarchies
	•	Runtime polymorphic surprises

Bestie enforces:
	•	Static dispatch by default
	•	Explicit polymorphism
	•	Ownership-aware object graphs
	•	Deterministic memory layout
	•	Compile-time resolvable semantics

Golden Rule

If behavior can be resolved at compile time, it must be resolved at compile time.

Any OOP feature violating this rule is excluded from the core language.

⸻

2. Class Kinds

Bestie supports multiple explicit class kinds, each with strict guarantees.

⸻

2.1 data class

Purpose
	•	Pure data aggregation
	•	Structural equality
	•	Domain modeling

Properties
	•	Fields only
	•	Immutable by default
	•	No identity semantics
	•	No inheritance
	•	No virtual methods

data class User {
    id: int
    name: str
}

Rules
	•	Cannot be open
	•	Cannot be inherited
	•	Cannot declare protected members
	•	Inner classes must be priv value class only

⸻

2.2 value class

Purpose
	•	Lightweight, inlineable objects
	•	Zero or near-zero overhead

Properties
	•	No identity
	•	Copy-by-value
	•	No inheritance
	•	No virtual dispatch

value class Point {
    x: int
    y: int
}

Rules
	•	Cannot contain own fields
	•	Cannot be open
	•	Inner classes are forbidden

⸻

2.3 enum / enum class

Purpose
	•	Closed sets of values
	•	Compile-time exhaustiveness

enum class Status {
    Active,
    Disabled
}

Rules
	•	No inheritance
	•	No mutable state
	•	Always thread-safe

⸻

2.4 single class

Purpose
	•	Process-level singleton
	•	Global coordination point

single class Config {
    port: int
}

Properties
	•	Exactly one instance per process
	•	Implicit, thread-safe access
	•	Lazy and deterministic initialization

Rules
	•	Cannot be open
	•	Cannot be inherited
	•	Inner classes must be priv
	•	Mutable state allowed (user responsibility)

⸻

2.5 class (Closed by Default)

Purpose
	•	Standard object with identity

class File {
    path: str
}

Properties
	•	Closed by default
	•	Static dispatch
	•	No inheritance unless explicitly enabled

⸻

2.6 open class

Purpose
	•	Explicit inheritance root
	•	Explicit polymorphic intent

open class Shape {
    @virtual fun area(): int
}

Rules
	•	Must be explicitly marked open
	•	Virtual methods must be explicitly annotated
	•	Single inheritance only

⸻

2.7 abstract class

Purpose
	•	Partial implementation
	•	Shared logic

Rules
	•	May contain abstract methods
	•	Cannot be instantiated
	•	Follows all open class rules

⸻

3. Visibility Modifiers

Bestie supports:

pub | pkg | protec | priv

Rules
	•	Top-level declarations cannot be priv
	•	pkg is the default
	•	protec applies only to inheritance
	•	Inner declarations cannot widen visibility

pkg class A {
    pub class B {}   // ❌ illegal
}


⸻

4. Inner Classes

Inner classes are lexically nested, not implicitly bound.

Rules
	•	No implicit capture of outer instance
	•	No hidden references
	•	Explicit qualification required
	•	Visibility constrained by outer declaration

Inner classes:
	•	May declare methods and properties
	•	May use annotations
	•	Cannot be open if outer is data, value, or single

⸻

5. this and super Resolution Rules

5.1 this

this always resolves statically.

Rules:
	•	Refers to the lexically enclosing instance
	•	Resolution is compile-time
	•	No implicit rebinding

class Outer {
    val x: int

    class Inner {
        fun f(o: Outer) {
            this      // Inner
            o          // Outer (explicit)
        }
    }
}

There is no implicit Outer.this.
All outer access must be explicit and type-safe.

⸻

5.2 super

super is allowed only when inheritance exists and is fully resolvable at compile time.

Rules:
	•	Refers to the immediate parent class
	•	Cannot be dynamic
	•	Cannot cross containment boundaries

open class A {
    fun f(): int
}

class B : A {
    fun g(): int {
        return super.f()
    }
}

Invalid cases:
	•	super in non-inheriting classes
	•	super inside inner classes
	•	super targeting protocol defaults directly

Protocol default resolution follows explicit rules (see Section 8).

⸻

6. Properties (Fields with Accessors)

Properties compile to explicit getter/setter methods.

val name: str => { get }
var age: int  => { get; set }

Rules
	•	No implicit backing fields
	•	Ownership rules apply
	•	Mutation must be explicit

Allowed in
	•	class
	•	open class
	•	single class
	•	Inner classes

Forbidden in
	•	protocol
	•	Discouraged in data / value classes

⸻

7. Protocols (Behavior Contracts)

Protocols define behavioral contracts, not state.

protocol Printable {
    fun print(): str
}

Rules
	•	No fields
	•	Methods only
	•	Default implementations allowed
	•	No instance state
	•	Static dispatch by default

⸻

7.1 Protocol Inheritance

Protocols may extend other protocols.

protocol Hashable {
    fun hash(): int
}

protocol Comparable {
    fun compare(other: Self): int
}

protocol Printable {
    fun print(): str
}

protocol Object : Hashable, Comparable, Printable

Rules
	•	All parent protocol methods are included
	•	Method resolution is compile-time
	•	No diamond ambiguity (no state, no fields)
	•	Conflicting defaults must be resolved explicitly by implementors

Protocols do not form runtime hierarchies.

⸻

8. Polymorphism Model

Bestie supports three explicit forms of polymorphism.

⸻

8.1 Overloading (Static)

fun draw(x: int)
fun draw(x: Point)

Resolved entirely at compile time.

⸻

8.2 Static Protocol Polymorphism (Default)

protocol Logger {
    fun log(msg: str)
}

	•	Compile-time dispatch
	•	No vtables
	•	Zero runtime cost

⸻

8.3 Dynamic Polymorphism (Explicit)

Dynamic dispatch requires @virtual.

protocol Shape {
    @virtual fun area(): int
}

Rules
	•	@virtual is mandatory
	•	@override is mandatory
	•	Vtables generated only when required
	•	Dynamic dispatch is opt-in and localized

⸻

9. Inheritance & Override Rules

9.1 Override Rules
	•	@override is mandatory
	•	Applies to static and virtual methods
	•	Signature must match exactly

⸻

9.2 Default Implementation Resolution

If:
	•	Class A extends class B
	•	Implements protocols C, D
	•	All define method m

Resolution order:
	1.	Class B
	2.	Concrete implementation in A
	3.	Protocol default implementations

Protocol defaults are never implicitly chained.

⸻

10. Construction Rules (init / new)

Bestie separates:
	•	Allocation (new)
	•	Initialization (init)

Rules
	•	Class.new() is canonical
	•	init() never allocates
	•	Builders/factories may hide both

Restrictions:
	•	@noNew
	•	@noInit
	•	@noConstruct

Enforced at compile time.

⸻

11. Design Patterns in Core

Included:
	•	single class (Singleton)

In std-lib:
	•	Factory (protocol)
	•	Builder (protocol)

Excluded:
	•	Observer
	•	Strategy
	•	Visitor
	•	Command

These are expressible via:
	•	Protocols
	•	Functions
	•	Lambdas

⸻

12. Thread Safety Guarantees

Always thread-safe:
	•	data class
	•	value class
	•	enum/enum class
	•	single class (initialization)
	•	@immutable classes
	•	Effectively immutable classes (all instance members are val)

User responsibility:
	•	open class
	•	Classes with var instance members

Concurrency safety is ownership-driven, not lock-driven.

⸻

13. What Bestie Deliberately Avoids
	•	Implicit virtual methods
	•	Multiple inheritance
	•	Runtime RTTI
	•	Reflection-based dispatch
	•	Fragile base classes

⸻

14. Summary

Bestie OOP is:
	•	Explicit
	•	Predictable
	•	Compile-time resolvable
	•	Memory-safe
	•	Concurrency-aware

OOP in Bestie exists to model reality clearly, not to enable accidental complexity.

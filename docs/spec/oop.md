# Bestie Language — Object-Oriented Programming (OOP)

This document defines **Object-Oriented Programming in Bestie**.

OOP in Bestie is:
- Explicit
- Compile-time driven
- Static by default
- Dynamic only when requested
- Memory-aware
- Concurrency-safe by design

Bestie treats OOP as a **tool**, not a paradigm.

---

## 1. Core OOP Philosophy

Bestie rejects:
- Implicit dynamic dispatch
- “Everything is an object”
- Inheritance-heavy design
- Runtime polymorphic surprises

Bestie enforces:
- Static dispatch by default
- Explicit polymorphism
- Ownership-aware object graphs
- Clear memory and concurrency semantics

If OOP exists in Bestie, it must:
1. Be analyzable at compile time
2. Preserve performance
3. Preserve memory layout guarantees
4. Not hide behavior

---

## 2. Class Kinds

Bestie supports the following **class kinds**:

### 2.1 `data class`

Purpose:
- Pure data aggregation
- Structural equality
- Domain modeling

Properties:
- Fields only
- No mutable state by default
- No inheritance
- No virtual methods

Example:
```bestie
data class User {
    id: int
    name: str
}

Rules:
	•	Cannot be open
	•	Cannot have protected members
	•	Inner classes must be priv value classes only

⸻

2.2 value class

Purpose:
	•	Lightweight, inlineable objects
	•	Zero or near-zero overhead

Properties:
	•	No identity
	•	Copy by value
	•	No inheritance
	•	No virtual dispatch

Example:
value class Point {
    x: int
    y: int
}

Rules:
	•	Cannot contain own fields
	•	Cannot be open
	•	Inner classes forbidden

⸻

2.3 enum / enum class

Purpose:
	•	Closed sets of values
	•	Compile-time exhaustiveness

Example:
enum class Status {
    Active,
    Disabled
}

Rules:
	•	No inheritance
	•	No mutable state
	•	Always thread-safe

⸻

2.4 single class

Purpose:
	•	Process-level singleton
	•	Global coordination point

Example:
single class Config {
    port: int
}

Properties:
	•	Exactly one instance per process
	•	getInstance() is implicit and thread-safe
	•	Lazy, safe initialization

Rules:
	•	Cannot be open
	•	Cannot be inherited
	•	Inner classes must be priv
	•	Mutable state allowed (user responsibility)

⸻

2.5 class (closed by default)

Purpose:
	•	Standard object with identity

Example:
class File {
    path: str
}

Properties:
	•	Closed by default
	•	No inheritance unless explicitly allowed
	•	Static dispatch

⸻

2.6 open class

Purpose:
	•	Explicit inheritance root
	•	Explicit polymorphic intent

Example:
open class Shape {
    @virtual fun area(): int
}

Rules:
	•	Must be explicitly marked open
	•	Virtual methods must be explicitly annotated
	•	Inheritance is single-only

⸻

2.7 abstract class

Purpose:
	•	Partial implementation
	•	Shared logic

Rules:
	•	May contain abstract methods
	•	Cannot be instantiated
	•	Follows same rules as open class

⸻

3. Visibility Modifiers

Bestie supports:
pub | pkg | protec | priv
Rules:
	•	Top-level classes cannot be priv
	•	pkg is the default
	•	protec only applies to inheritance
	•	Inner members cannot widen visibility beyond their outer scope

Example (illegal):
pkg class A {
    pub class B {}   // ❌ illegal
}

⸻

4. Inner Classes

Inner classes are lexically nested, not implicitly bound.

Rules:
	•	No implicit access to outer instance
	•	No hidden captures
	•	Must obey outer visibility

Inner classes may have:
	•	Methods
	•	Properties
	•	Annotations

Inner classes may not:
	•	Be open if outer is single or data
	•	Escalate visibility

⸻

5. Properties (Fields with Accessors)

Property syntax:
val name: str => { get }
var age: int => { get; set }

Rules:
	•	Properties compile to methods
	•	No backing field magic
	•	Ownership rules apply

Ownership example:
val own address: Address => { get }
Properties:
	•	Allowed in classes and single classes
	•	Allowed in inner classes
	•	Forbidden in protocols
	•	Discouraged in data/value classes (prefer direct fields)

⸻

6. Protocols (Interfaces)

Protocols define behavior contracts.

Example:
protocol Serializable {
    fun serialize(): str
}
Rules:
	•	No fields
	•	Methods only
	•	Default implementations allowed
	•	No state
	•	Static dispatch by default

⸻

7. Group Protocols

Group protocols aggregate protocols:
group protocol Persistable {
    Readable,
    Writable
}
Rules:
	•	No methods of their own
	•	No state
	•	Dispatch rules inherited from members
	•	May mix static and virtual protocols

⸻

8. Polymorphism Model

Bestie supports three forms of polymorphism:

8.1 Overloading (Static)
fun draw(x: int)
fun draw(x: Point)

Resolved at compile time.

⸻

8.2 Static Protocol Polymorphism (Default)
protocol Logger {
    fun log(msg: str)
}

Dispatch:
	•	Compile-time
	•	No vtables
	•	Zero overhead

⸻

8.3 Dynamic Polymorphism (Explicit)

Dynamic dispatch requires @virtual.
protocol Shape {
    @virtual fun area(): int
}

Rules:
	•	@virtual must be explicit
	•	@override required
	•	Vtables only generated when needed

⸻

9. Inheritance & Override Rules

9.1 Override Rules
	•	@override is mandatory
	•	Applies to both static and virtual methods
	•	Signature must match exactly

⸻

9.2 Default Implementation Resolution

If:
	•	Class A extends class B
	•	Implements protocols C, D
	•	All define method m

Then:
	1.	Class B implementation wins
	2.	If B.m is abstract → A must implement
	3.	Protocol defaults are secondary

This rule is non-negotiable.

⸻

10. Construction Rules (init / new)

Bestie separates:
	•	Initialization (init)
	•	Allocation (new)

Rules:
	•	Class.new() is canonical
	•	init() never allocates
	•	Factories/builders should hide both

Preventing misuse:
	•	@noNew
	•	@noInit
	•	@noConstruct

These annotations may be applied to:
	•	Classes
	•	Protocols

Enforced at compile time.

⸻

11. Design Patterns in OOP Core

Included in core:
	•	single class (Singleton)

Included in std-lib:
	•	Factory (protocol)
	•	Builder (protocol)

Excluded:
	•	Observer
	•	Strategy
	•	Command
	•	Visitor

These are expressible using:
	•	Protocols
	•	Functions
	•	Lambdas

⸻

12. Thread Safety Guarantees

Always thread-safe:
	•	data class
	•	value class
	•	enum
	•	single class (initialization)
	•	@immutable classes

User responsibility:
	•	open class
	•	Mutable closed classes

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
	•	Fast
	•	Memory-safe
	•	Concurrency-aware

OOP exists to model reality, not to impress frameworks.


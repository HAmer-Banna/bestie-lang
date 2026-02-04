# Bestie Language — Object-Oriented Programming (OOP)

This document defines **Object-Oriented Programming in Bestie**.

OOP in Bestie is:

* Explicit
* Compile-time driven
* Static by default
* Dynamic only when explicitly requested
* Memory-aware
* Concurrency-safe by construction

Bestie treats OOP as a **controlled toolset**, not a dominant paradigm.

---

## 1. Core OOP Philosophy

Bestie explicitly rejects:

* Implicit dynamic dispatch
* “Everything is an object”
* Inheritance-heavy hierarchies
* Runtime polymorphic surprises

Bestie enforces:

* Static dispatch by default
* Explicit polymorphism
* Ownership-aware object graphs
* Deterministic memory layout
* Compile-time resolvable semantics

**Golden Rule:**
If behavior can be resolved at compile time, it **must** be resolved at compile time.

Any OOP feature violating this rule is excluded from the **core language**.

---

## 2. Polymorphism Model (Binding First)

Bestie supports **both static and dynamic polymorphism**, but they are **explicitly separated**.

Three forms exist:

### 2.1 Overloading (Static)

```bestie
fun draw(x: int)
fun draw(x: Point)
```

* Resolved entirely at compile time
* No runtime cost

---

### 2.2 Static Protocol Polymorphism (Default)

* Compile-time dispatch
* No vtables
* No RTTI
* Zero runtime cost

This is **early binding**.

---

### 2.3 Dynamic Polymorphism (Explicit)

* Requires `@virtual`
* `@override` mandatory
* Vtables generated only when required
* Dispatch happens at runtime

This is **late binding**, and is **opt-in only**.

---

## 3. Class Kinds

Bestie supports **multiple explicit class kinds**, each with strict guarantees.

---

### 3.1 data class

**Purpose:**

* Pure data aggregation
* Structural equality
* Domain modeling

**Properties:**

* Fields only
* Immutable by default
* No identity semantics
* No inheritance
* No virtual methods

```bestie
data class User {
    id: int
    name: str
}
```

**Rules:**

* Cannot be open or inherited
* Cannot declare `protec` members
* Inner classes must be `priv value class` only

---

### 3.2 value class

**Purpose:**

* Lightweight, inlineable objects
* Zero or near-zero overhead

**Properties:**

* No identity
* Copy-by-value
* No inheritance
* No virtual dispatch

```bestie
value class Point {
    x: int
    y: int
}
```

**Rules:**

* Cannot contain `own` fields
* Cannot be open
* Inner classes forbidden

---

### 3.3 enum / enum class

**Purpose:**

* Closed sets of values
* Compile-time exhaustiveness

```bestie
enum class Status {
    Active,
    Disabled
}
```

**Rules:**

* No inheritance
* No mutable state
* Always thread-safe

---

### 3.4 single class

**Purpose:**

* Process-level singleton
* Global coordination point

```bestie
single class Config {
    port: int
}
```

**Properties:**

* Exactly one instance per process
* Implicit, thread-safe access
* Lazy and deterministic initialization

**Rules:**

* Cannot be open or inherited
* Inner classes must be `priv`
* Mutable state.

---

### 3.5 class

**Purpose:**
Standard object with identity

```bestie
class File {
    path: str
}
```

**Properties:**

* Closed
* Static dispatch
* Cannot be inherited
* Can extend another class or implements protocol.

---

### 3.6 open class

**Purpose:**
Explicit inheritance root, explicit polymorphic intent

```bestie
open class Shape {
    @virtual fun area(): int
}
```

**Rules:**

* Must be explicitly marked `open`
* Virtual methods must be explicitly annotated with `@virtual`
* `@override` mandatory
* Single inheritance only
* open class that is not subclassed will trigger a compiler warning, discouraging the use of openness unless inheritance is explicitly intended

---

### 3.7 abstract class

**Purpose:**
Partial implementation with shared logic

**Rules:**

* May contain abstract methods
* Cannot be instantiated
* Follows all open class rules

---

** data, value, enum, and @immutable classes cannot extend other classes, but may implement protocols. Only statically dispatched (non-@virtual) protocol methods are permitted.

---

## 4. Protocols (Behavior Contracts)

Protocols define **behavior only**, not state.

```bestie
protocol Printable {
    fun print(): str
}
```

**Rules:**

* No fields
* Methods only
* Default implementations allowed
* No instance state
* Static dispatch by default

---

### 4.1 Protocol Inheritance

```bestie
protocol Object : Hashable, Comparable, Printable
```

**Rules:**

* All parent protocol methods are included
* Method resolution is compile-time
* No diamond ambiguity
* Conflicting defaults must be resolved explicitly

Protocols **do not form runtime hierarchies**.

---

## 5. Casting Rules (Classes & Protocols)

Casting in Bestie is **explicit, directional, and binding-aware**.

### 5.1 Upcasting (Implicit, Safe)

Allowed when moving **from concrete to abstraction**.

```bestie
val s: Shape = Circle.new()
val p: Printable = Circle.new()
```

* Always safe
* Compile-time verified
* Preserves early binding unless `@virtual` is involved

---

### 5.2 Downcasting (Explicit, Checked)

Required when moving **from abstraction to concrete**.

```bestie
val c: Circle = s as Circle
```

Rules:

* Must use `as`
* Fails at compile time if statically impossible
* Runtime check only exists if dynamic polymorphism is involved

---

### 5.3 Class ↔ Protocol Casting

```bestie
val p: Printable = Circle.new()
val c: Circle = p as Circle
```

Rules:

* Upcast to protocol is implicit
* Downcast requires `as`
* No RTTI-based discovery
* Validity is known at compile time for sealed protocols

---

### 5.4 Binding Preservation Rules

* Static calls remain statically bound after casting
* Dynamic dispatch only occurs for `@virtual` methods
* Casting **never introduces late binding**

---

## 6. Inheritance & Override Rules

* `@override` mandatory
* Signature must match exactly

### 6.1 Default Implementation Resolution

Order:

1. Parent class
2. Concrete subclass implementation
3. Protocol default implementation

Protocol defaults are **never implicitly chained**.

---

** Inheritance in Bestie is explicit: ext is used to extend a single class, and impl is used to implement one or more protocols.

---

## 7. Visibility Modifiers

Supported:

* `pub` | `pkg` | `protec` | `priv`

**Rules:**

* Top-level declarations cannot be `priv`
* `pkg` is default
* `protec` applies only to inheritance
* Inner declarations cannot widen visibility

---

## 8. Inner Classes

Inner classes are **lexically nested**, not implicitly bound.

Rules:

* No implicit capture of outer instance
* No hidden references
* Explicit qualification required
* Visibility constrained by outer declaration

---

## 9. `this` and `super` Resolution

### 9.1 `this`

* Always resolves statically
* Refers to lexically enclosing instance
* No implicit rebinding

---

### 9.2 `super`

* Compile-time resolvable only
* Refers to immediate parent
* Forbidden in inner classes and protocols

---

## 10. Properties (Fields with Accessors)

* Compile to explicit getter/setter methods
* No implicit backing fields
* Ownership rules apply

---

## 11. Construction Rules (`init` / `new`)

* `new()` allocates
* `init()` initializes
* Enforced at compile time
* Builders/factories may wrap both

Restrictions:
* `@noNew`
* `@noInit`
* `@noConstruct`

---

## 12. Sealed Declarations (File-Scoped)

Sealing defines **closed, finite sets** of declarations.

Applies to:

* Classes
* Protocols
* File-level functions

No runtime impact.

### 12.1 Sealed Classes

A sealed declaration explicitly enumerates which classes may extend a base class.

sealed Shape permits Circle, Rectangle, Triangle


open class Shape
class Circle : Shape
class Rectangle : Shape
class Triangle : Shape

Properties:

Only listed classes may extend the sealed base

All permitted classes must be in the same file

Enables exhaustive match checking

No runtime registration or reflection

Rules:

Base must be open or abstract

No implicit inheritance outside the permit list

Sealing does not affect dispatch or memory layout

### 12.2 Sealed Protocols

Protocols may also be sealed with an explicit implementor list.

sealed protocol Token permits NumberToken, StringToken


class NumberToken : Token
class StringToken : Token

Properties:

Only permitted types may implement the protocol

All implementors are known at compile time

Enables exhaustive protocol-based matching

Rules:

Protocol rules still apply (no state, no fields)

Default implementations remain static

### 12.3 Sealed File-Level Functions

File-level functions may be sealed to define a closed overload set.

sealed fun parse(str: str): Token
sealed fun parse(int: int): Token

Properties:

Only declared overloads are allowed

No external extension or overloading

Resolution remains compile-time

Rules:

All sealed overloads must appear in the same file

Prevents accidental API extension

---

## 13. Thread Safety Guarantees

Always thread-safe:

* data class
* value class
* enum / enum class
* single class initialization
* immutable classes

User responsibility:

* open classes
* mutable state

---

## 14. What Bestie Deliberately Avoids

* Implicit virtual methods
* Multiple inheritance
* Runtime RTTI
* Reflection-based dispatch
* Fragile base classes

---

## 15. Summary

Bestie OOP is a **compiler-driven object model** with:

* Early binding by default
* Late binding only by explicit request
* Explicit casting with preserved semantics
* No hidden runtime behavior
* Strong memory and concurrency guarantees

Classes model **identity and structure**.
Protocols model **behavior and capability**.
Casting never changes binding rules.

OOP in Bestie exists to **serve performance, predictability, and correctness** —
never abstraction for its own sake.

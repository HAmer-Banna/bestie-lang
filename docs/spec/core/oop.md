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

**Golden Rule:** If behavior can be resolved at compile time, it **must** be resolved at compile time.

Any OOP feature violating this rule is excluded from the **core language**.

---

## 2. Class Kinds

Bestie supports **multiple explicit class kinds**, each with strict guarantees.

---

### 2.1 data class

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

### 2.2 value class

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

### 2.3 enum / enum class

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

### 2.4 single class

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
* Mutable state allowed (user responsibility)

---

### 2.5 class (Closed by Default)

**Purpose:** Standard object with identity

```bestie
class File {
    path: str
}
```

**Properties:**

* Closed by default
* Static dispatch
* No inheritance unless explicitly enabled

---

### 2.6 open class

**Purpose:** Explicit inheritance root, explicit polymorphic intent

```bestie
open class Shape {
    @virtual fun area(): int
}
```

**Rules:**

* Must be explicitly marked `open`
* Virtual methods must be explicitly annotated with `@virtual`
* `@override` mandatory for overriding
* Single inheritance only

---

### 2.7 abstract class

**Purpose:** Partial implementation with shared logic

**Rules:**

* May contain abstract methods
* Cannot be instantiated
* Follows all open class rules

---

## 3. Visibility Modifiers

Supported:

* `pub` | `pkg` | `protec` | `priv`

**Rules:**

* Top-level declarations cannot be `priv`
* `pkg` is default
* `protec` applies only to inheritance
* Inner declarations cannot widen visibility

```bestie
pkg class A {
    pub class B {}   // ❌ illegal
}
```

---

## 4. Inner Classes

Inner classes are **lexically nested**, not implicitly bound.

**Rules:**

* No implicit capture of outer instance
* No hidden references
* Explicit qualification required
* Visibility constrained by outer declaration

Inner classes may:

* Declare methods and properties
* Use annotations
* Cannot be open if outer is data, value, or single

---

## 5. `this` and `super` Resolution

### 5.1 `this`

* Always resolves statically
* Refers to **lexically enclosing instance**
* No implicit rebinding

```bestie
class Outer {
    val x: int
    class Inner {
        fun f(o: Outer) {
            this      // Inner
            o          // Outer (explicit)
        }
    }
}
```

No implicit `Outer.this`. All outer access must be explicit.

### 5.2 `super`

* Allowed only when inheritance exists and is fully resolvable at compile time
* Refers to **immediate parent class**
* Cannot be dynamic or cross containment boundaries

```bestie
open class A { fun f(): int }
class B : A {
    fun g(): int { return super.f() }
}
```

Invalid cases:

* `super` in non-inheriting classes
* `super` inside inner classes
* `super` targeting protocol defaults directly

---

## 6. Properties (Fields with Accessors)

* Compile to explicit getter/setter methods

```bestie
val name: str => { get }
var age: int => { get; set }
```

**Rules:**

* No implicit backing fields
* Ownership rules apply
* Mutation must be explicit

Allowed in:

* `class`, `open class`, `single class`, inner classes

Forbidden in:

* `protocol`
* Discouraged in data/value classes

---

## 7. Protocols (Behavior Contracts)

* Define **behavior**, not state

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

### 7.1 Protocol Inheritance

```bestie
protocol Object : Hashable, Comparable, Printable
```

**Rules:**

* All parent protocol methods are included
* Method resolution is **compile-time**
* No diamond ambiguity (no state, no fields)
* Conflicting defaults must be resolved explicitly by implementors

Protocols **do not form runtime hierarchies**.

---

## 8. Polymorphism Model

Three explicit forms of polymorphism:

### 8.1 Overloading (Static)

```bestie
fun draw(x: int)
fun draw(x: Point)
```

* Resolved entirely at compile time

### 8.2 Static Protocol Polymorphism (Default)

* Compile-time dispatch
* No vtables
* Zero runtime cost

### 8.3 Dynamic Polymorphism (Explicit)

* Requires `@virtual` annotation
* `@override` is mandatory
* Vtables generated only when required
* Dynamic dispatch is **opt-in**

---

## 9. Inheritance & Override Rules

* `@override` mandatory for all overridden methods
* Signature must match exactly

### 9.1 Default Implementation Resolution

Order:

1. Parent class
2. Concrete implementation in subclass
3. Protocol default implementations

*Protocol defaults are never implicitly chained.*

---

## 10. Construction Rules (`init` / `new`)

* `new()` = allocation
* `init()` = initialization
* Builders/factories may wrap both

Restrictions:

* `@noNew`
* `@noInit`
* `@noConstruct`

Enforced **at compile time**.

---

## 11. Design Patterns in Core

Included:

* `single class` (Singleton)

In std-lib:

* `Factory` (protocol)
* `Builder` (protocol)

Excluded (expressible via protocols/lambdas):

* Observer, Strategy, Visitor, Command

---

## 12. Thread Safety Guarantees

Always thread-safe:

* data class, value class, enum/enum class, single class (initialization), `@immutable` classes, effectively immutable classes (all `val`)

User responsibility:

* open class
* Classes with mutable (`var`) members

*Concurrency safety is ownership-driven, not lock-driven.*

---

## 13. What Bestie Deliberately Avoids

* Implicit virtual methods
* Multiple inheritance
* Runtime RTTI
* Reflection-based dispatch
* Fragile base classes

---

## 14. Summary

Bestie OOP is:

* Explicit
* Predictable
* Compile-time resolvable
* Memory-safe
* Concurrency-aware

OOP exists to **model reality clearly**, not to enable accidental complexity.


Do you want me to create that diagram next?

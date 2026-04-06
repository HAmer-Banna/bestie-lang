# Bestie Language — Object-Oriented Programming (OOP)

This document defines **Object-Oriented Programming in Bestie**.

OOP in Bestie is:

* Explicit
* Compile-time driven
* Static by default
* Dynamic only when explicitly requested
* Memory-aware
* Concurrency-safe for ownership-validated code paths

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

Bestie guarantees the best memory layout.

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

Bestie has one class keyword with semantic modifiers.
Bestie supports **multiple explicit class shapes**, each with strict guarantees.

---

### 3.1 data class

**Purpose:**

* Pure data aggregation
* Structural equality
* Domain modeling

**Properties:**

* Fields only — all fields are `val`, always
* Deeply immutable — no mutable state anywhere in the graph
* No identity semantics
* No inheritance
* No virtual methods
* Always thread-safe

```bestie
data class User {
    id: int
    name: str
}
```

**Rules:**

* All fields are implicitly `val` — `var` fields are **forbidden**
* Fields holding collections must use `.immutable` collections
* Cannot be open or inherited
* Cannot declare `protected` members
* Inner classes must be `private value class` only
* Cannot use `ext`
* May use `impl` with protocols (static dispatch only)

If you need mutable fields, use a regular `class` instead.

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

### 3.3 enum

**Purpose:**

In Bestie, `enum` is one core shape with two declaration forms:

```bestie
enum WeekendDays {
    FRIDAY,
    SATURDAY
}
```

For richer cases, the same `enum` keyword supports payload variants, generics (`<T>`), and protocol `impl` (static dispatch only):

```bestie
enum Status<T> {
    Active(T),
    Disabled(str)
}
```

* Closed sets of values
* Compile-time exhaustiveness
* Value-type semantics by default

**Rules:**

* No inheritance
* No `ext`
* No mutable state
* No hidden heap allocation requirement
* Tag-only enums lower to compact integer tags
* Always thread-safe

---

### 3.4 class

**Purpose:**
Standard object with identity

```bestie
class File {
    path: str
}
```

**Properties:**

* Closed by default
* Static dispatch
* A `class` can be a subclass, but not an inheritance root
* May use `ext` with an `open class` or `abstract class` base
* May use `impl` with one or more protocols

---

### 3.5 open class

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

**Warning — unsubclassed `open` within module:**

An `internal open class` with no subclasses anywhere in the module triggers a **compiler warning**:

```
warning: 'Shape' is declared open but has no subclasses — consider removing 'open'
```

The intent is to discourage the habit of marking classes open preemptively ("just in case we need to subclass it later"). `open` is a deliberate design decision, not a default safety net.

`public open class` and `protected open class` do **not** trigger this warning — external modules or subclasses within the hierarchy may extend them, and the compiler cannot know at module compile time.

---

### 3.6 abstract class

**Purpose:**
Partial implementation with shared logic

**Rules:**

* May contain abstract methods
* Cannot be instantiated
* Follows all open class rules

---

`data class`, `value class`, `enum`, and `@immutable class` cannot use `ext`.
They may use `impl` with protocols, but only statically dispatched protocol methods are allowed.

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
protocol Object ext Hashable, Comparable, Printable
```

**Rules:**

* All parent protocol methods are included
* Method resolution is compile-time
* Protocols **do not form runtime hierarchies**

---

### 4.2 Default Method Conflict Resolution

Conflict resolution follows the same rules as Java interfaces:

---

**Rule 1 — Class implementation always wins:**

A concrete implementation in the class takes priority over any protocol default, regardless of how many protocols are involved.

```bestie
protocol A { fun greet(): str = "A" }

class C impl A {
    fun greet(): str = "C"    // wins — class beats protocol default
}
```

---

**Rule 2 — More specific protocol wins:**

If protocol `B ext A` overrides a default from `A`, `B`'s version takes priority when a class implements `B`.

```bestie
protocol A { fun greet(): str = "A" }
protocol B ext A { fun greet(): str = "B" }   // more specific

class C impl B {
    // no override needed — B.greet wins over A.greet
}
```

---

**Rule 3 — Ambiguous conflict requires explicit resolution:**

When two independent protocols provide conflicting defaults and neither is more specific, the implementing class **must** explicitly override and resolve by calling the desired protocol's default via `Protocol.method(this)`:

```bestie
protocol A { fun greet(): str = "A" }
protocol B { fun greet(): str = "B" }

class C impl A, B {
    @override
    fun greet(): str = A.greet(this)    // explicit — pick A's default
}
```

Without the explicit override, the compiler emits a **compile-time error**:

```
error: 'C' inherits conflicting defaults for 'greet' from protocols A and B — must override explicitly
```

---

**Rule 4 — Class hierarchy beats protocol defaults:**

A method inherited from a parent class takes priority over a protocol default, even if the protocol is implemented lower in the hierarchy.

```bestie
open class Base {
    fun greet(): str = "Base"
}

protocol A { fun greet(): str = "A" }

class Child ext Base impl A {
    // Base.greet wins — no override needed, no conflict
}
```

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
* Runtime check exists only for `@virtual` hierarchies
* Runtime check uses compiler-emitted type metadata, not reflection APIs

---

### 5.3 Class ↔ Protocol Casting

```bestie
val p: Printable = Circle.new()
val c: Circle = p as Circle
```

Rules:

* Upcast to protocol is implicit
* Downcast requires `as`
* No reflection-based type discovery
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
* Only `open` and `abstract` classes are inheritable
* A closed `class` cannot be extended
* A closed `class` may extend an `open` or `abstract` base

### 6.1 Default Implementation Resolution

Priority order (highest to lowest):

1. Concrete implementation in the class itself
2. Method inherited from the parent class
3. More specific protocol default (protocol that extends another)
4. Protocol default implementation

When two protocol defaults are equally specific and conflict, explicit resolution is required — see section 4.2 Rule 3.

Protocol defaults are **never implicitly chained**.

---

Inheritance in Bestie is explicit:

* `ext` extends exactly one class
* `impl` implements one or more protocols

---

## 7. Visibility Modifiers

| Modifier | Scope                                             |
| -------- | ------------------------------------------------- |
| `public`    | Visible outside the module (exported API)         |
| `internal`    | Visible anywhere within the same module (default) |
| `protected` | Visible to subclasses only                        |
| `private`   | Visible inside the declaring type only            |

**Rules:**

* `internal` is the default — no modifier means `internal`
* Top-level declarations cannot be `private`
* `protected` applies only within inheritance hierarchies
* Inner declarations cannot widen visibility beyond their enclosing declaration

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

A sealed declaration explicitly enumerates which classes may `ext` a base class.

```bestie
sealed Shape permits Circle, Rectangle, Triangle

open class Shape
class Circle ext Shape
class Rectangle ext Shape
class Triangle ext Shape
```

Properties:

* Only listed classes may `ext` the sealed base
* All permitted classes must be in the same file
* Enables exhaustive match checking
* No runtime registration or reflection

Rules:
* Base must be open or abstract
* No implicit inheritance outside the permit list
* Sealing does not affect dispatch or memory layout

### 12.2 Sealed Protocols

Protocols may also be sealed with an explicit `impl` list.

```bestie
sealed protocol Token permits NumberToken, StringToken

class NumberToken impl Token
class StringToken impl Token
```

Properties:

* Only permitted types may `impl` the protocol
* All implementors are known at compile time
* Enables exhaustive protocol-based matching

Rules:

* Protocol rules still apply (no state, no fields)
* Default implementations remain static

### 12.3 Sealed File-Level Functions

File-level functions may be sealed to define a closed overload set.

`sealed fun parse(str: str): Token`
`sealed fun parse(int: int): Token`

Properties:

* Only declared overloads are allowed
* No external extension or overloading
* Resolution remains compile-time

Rules:

* All sealed overloads must appear in the same file
* Prevents accidental API extension

---

## 13. Thread Safety Guarantees

Thread safety comes from the **type**, not the binding. `val` makes a binding immutable — it does not make the value deeply immutable or safe to share across threads.

Always thread-safe — deep immutability is declared explicitly:

* `data class` — value semantics, no identity
* `value class` — value semantics, no identity
* `enum` — closed, no mutable state
* Classes annotated `@immutable` — deep immutability enforced by the compiler
* Primitive types (`int`, `float64`, `bool`, `char`, etc.)
* Immutable collections — created via `.immutable` builder: `list<T>.immutable.build()`, `set<T>.immutable.build()`, etc.
* `const` values — stored in read-only memory

**Not** automatically thread-safe:

* `val xs: list<int>` — the binding is immutable, but the list is mutable
* A class with all `val` fields, if any field holds a mutable collection or a mutable class
* `open class` and regular `class` instances unless annotated `@immutable`

User responsibility:

* `open class` and mutable classes — use locks or ownership transfer
* Mutable collections — use `.concurrent` builder or ownership transfer
* Singleton-style global objects — use atomics or locks from std-api

---

## 14. What Bestie Deliberately Avoids

* Implicit virtual methods
* Multiple inheritance
* General-purpose runtime RTTI APIs
* Reflection-based dispatch
* Fragile base classes
* Language-level singleton types

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

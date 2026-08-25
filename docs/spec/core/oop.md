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
* Vtables generated only for open `@virtual` hierarchies
* Sealed `@virtual` hierarchies may use compact tags and direct dispatch instead of vtables
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

**Construction:**

* Always receives a compiler-generated memberwise `init()` (see section 11.4)
* `init()` may not be fallible — `data class` construction cannot return `!`
* `@virtual` calls in `init()` are impossible (no virtual methods permitted)
* `@noInit` suppresses the generated init; `@noConstruct` forbids all construction

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

**Construction:**

* Always receives a compiler-generated memberwise `init()` (see section 11.4)
* No `super.init(...)` — `value class` cannot use `ext`
* Because `own` fields are forbidden, no ownership drops are emitted for partial init failure
* Laid out inline when used as a field of another class (see section 11.9)

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

**Construction:**

* Receives a compiler-generated memberwise `init()` if none is declared (see section 11.4)
* If using `ext`, every `init()` must call `super.init(...)` exactly once, optionally preceded by a `this`-free prologue (see section 11.3)
* May have fallible `init()` returning `! ErrorSet` (see section 11.6)
* `@virtual` calls inside `init()` are a compile-time error (see section 11.7)

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

**Construction:**

* `init()` follows standard rules (see section 11)
* Subclasses must call `super.init(...)` exactly once in every `init()`, optionally preceded by a `this`-free prologue (see section 11.3)
* `@virtual` methods must not be called from `init()` — the derived subclass is not yet initialized (see section 11.7)

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

**Construction:**

* May declare `init()` with shared initialization logic
* `abstract class` `init()` is only reachable via `super.init(...)` from a concrete subclass
* Constructing an abstract class (`T(...).new()`) is a compile-time error — instantiation is forbidden

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

### 4.3 Implementing Protocols (No Anonymous Classes)

Bestie has **no anonymous classes** — there is no inline `impl Protocol { ... }` object expression and no on-the-fly subclass. Every implementer is a **named** type.

* **Multi-method protocols** are implemented by a named class. For locality near the use site, that class may be file-private or an inner class (section 8); it remains named, statically dispatched, and compile-time laid out.
* **Single-abstract-method (SAM) protocols** may instead receive a **lambda** directly — the compiler synthesizes the implementer at compile time. See `core/fp.md` §16.2 for the exact SAM conversion rules.

This keeps every behavior contract backed by either a named type or a lambda value, with no unnamed runtime types. The anonymous construct in Bestie is the **lambda**, not the class.

---

## 5. Casting Rules (Classes & Protocols)

Casting in Bestie is **explicit, directional, and binding-aware**.

### 5.1 Upcasting (Implicit, Safe)

Allowed when moving **from concrete to abstraction**.

```bestie
val s: Shape = Circle().new()
val p: Printable = Circle().new()
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
* Sealed hierarchies may lower the check to a compact type-tag test
* Runtime check uses compiler-emitted type metadata, not reflection APIs

---

### 5.3 Class ↔ Protocol Casting

```bestie
val p: Printable = Circle().new()
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

### 5.5 Type Checking with `is` (instanceof-equivalent)

`is` is Bestie's runtime type-test operator. It is the equivalent of Java's `instanceof`: it answers the question *"does this value's actual type conform to `T`?"* and evaluates to a `bool`.

```bestie
val matched: bool = shape is Circle
```

**Where it is meaningful:**

`is` only performs a *runtime* test where a runtime type actually exists:

* `@virtual` hierarchies (`open` / `abstract class`) — the check reads compiler-emitted type metadata.
* Sealed hierarchies — the check lowers to a compact type-tag comparison (no metadata pointer walk).

In both cases the test uses compiler-emitted type information, **never reflection or a general RTTI API**.

**Static (compile-time) resolution:**

For a value whose concrete type is fully known at compile time (no `@virtual` involved), `x is T` is resolved at compile time to a constant `true` or `false`. Because static dispatch already knows the type, a check that can never vary is redundant — the compiler emits a warning:

```
warning: 'x is Circle' is always true — type is statically known, 'is' check is redundant
```

**Pairing with `as` (test-then-cast):**

`is` narrows knowledge; `as` performs the cast. The idiomatic pattern is to test with `is`, then downcast with `as`:

```bestie
open class Shape {
    @virtual fun area(): float64
}
class Circle ext Shape { radius: float64 }
class Rectangle ext Shape { width: float64; height: float64 }

fun describe(s: Shape): str {
    if (s is Circle) {
        val c = s as Circle          // safe — the runtime type was just tested
        return "circle r=${c.radius}"
    }
    if (s is Rectangle) {
        val r = s as Rectangle
        return "rect ${r.width}x${r.height}"
    }
    return "unknown shape"
}
```

**Rules:**

* `is` respects the hierarchy: `child is Base` is `true` when `child`'s runtime type derives from `Base` (matching `instanceof` semantics).
* Negation uses the standard logical operators: `not (s is Circle)` or `!(s is Circle)`.
* `is` against an unrelated type that can never match is a **compile-time error**, not a runtime `false` (e.g. `circle is String` where the types share no hierarchy).
* For sealed hierarchies, prefer exhaustive `switch`/pattern matching (section 12) over chains of `is`; the compiler can then verify all cases are handled.
* `is` introduces no late binding and no reflection — it is purely a type-tag / metadata comparison resolved by the compiler.

---

## 6. Inheritance & Override Rules

* `@override` mandatory
* Signature must match exactly
* Only `open` and `abstract` classes are inheritable
* A closed `class` cannot be extended
* A closed `class` may extend an `open` or `abstract` base

Overriding is distinct from lexical shadowing: it replaces a `@virtual` method's implementation and is resolved by dynamic dispatch, whereas shadowing only hides a binding in a nested scope at compile time. Bestie has **no member hiding** — a subclass may not redeclare an inherited field or non-`@virtual` method. See `core/base.md` §4.8.4 for the shadowing-vs-overriding distinction.

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

For construction rules specific to inner classes — including how the outer `init()` owns and sequences inner construction, and how fallible inner init propagates — see section 11.9.

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
* `super.init(...)` must appear exactly once in any derived class `init()`, preceded only by a `this`-free prologue — see section 11.3

---

## 10. Properties (Fields with Accessors)

Properties expose accessor-backed values that look like fields at the call site but compile to explicit getter/setter methods. There is no implicit backing field — the accessors are the property.

**Syntax:**

```bestie
val name: str => { get }
var age: int => { get; set }
```

* `val` properties expose a `get` accessor only (read-only).
* `var` properties expose both `get` and `set` accessors (read/write).

**Rules:**

* Properties compile to methods — there is no backing field magic
* No implicit backing fields are generated
* Ownership rules apply

**Ownership example:**

```bestie
val own address: Address => { get }
```

**Accessors with a body (`get` / `set` logic):**

The shorthand forms above declare accessors with no logic. When a property needs to compute a value, validate an assignment, or wrap an explicit field, each accessor may instead be given an explicit **body**. Because Bestie generates no implicit backing field, a bodied accessor operates over state the class declares itself — a bodied property is either **computed** (derived from other fields) or a **validating wrapper** around an explicit private field.

```bestie
class Temperature {
    private var _celsius: float64

    // read-only computed property — a `get` body derives the value, no stored field of its own
    val fahrenheit: float64 => {
        get { return this._celsius * 9.0 / 5.0 + 32.0 }
    }

    // read/write property — `get` and `set` bodies wrap an explicit field
    var celsius: float64 => {
        get { return this._celsius }
        set(value) {
            if (value < -273.15) { panic("temperature below absolute zero") }
            this._celsius = value
        }
    }
}
```

**Accessor body rules:**

* The `get` body must `return` a value of the property's declared type.
* The `set` body receives the incoming value as a parameter — `set(value) { ... }`. The parameter name is chosen by the author (`set(v)` is equally valid).
* A `val` property may declare a `get` body only. A `var` property may declare both a `get` and a `set` body.
* Bodied accessors reference explicit fields — there is **no implicit backing field**. A computed property that only reads other fields declares no field of its own; a stored/validated property declares its own (typically `private`) field.
* An accessor body follows normal method rules (ownership, visibility, static dispatch). Accessors are **not** `@virtual`.
* Shorthand (`=> { get }` / `=> { get; set }`) and bodied accessors may be mixed within the same class as appropriate — use the shorthand for trivial exposure and a body only where logic is required.

**Allowed contexts:**

* Allowed in classes and singleton classes
* Allowed in inner classes
* Forbidden in protocols
* Discouraged in `data`/`value` classes (prefer direct fields)

---

## 11. Construction & Destruction Rules (`init` / `deinit`)

### 11.1 Two-Phase Construction Model

Construction is written as two explicit, chained steps — **nothing is called implicitly**:

```bestie
Connection("localhost", 9000).new()
```

1. **`Type(args)` — the initializer expression.** Selects an `init(...)` overload declared on the type (§11.5) and binds its arguments. The call site does **not** write `.init` — `Type(...)` *is* the init selection. It is a **compile-time construction expression, not a value**: it cannot be bound to a variable, stored in a field, passed to a function, or returned. Its only valid continuation is a materialization verb.
2. **`.new()` — materialization.** Takes **no arguments**. It allocates storage (heap for reference classes, yielding an `own Type`; stack/inline for `value` and `data` classes) and then runs the construction work: the prologue (§11.3), the `super.init(...)` chain, and field initialization (§11.2).

Both steps together form **one atomic construction** from the caller's perspective. A partially-constructed object is **never observable** by user code — because `Type(args)` is not itself an object, there is no uninitialized instance to observe. The caller receives either a fully-initialized object or an error (§11.6).

`init` remains the **declaration** keyword inside the class body, and `super.init(...)` / `this.init(...)` remain the **internal** delegation forms (§11.3, §11.5). Only the external call site omits the word `init`, to avoid redundant verbosity.

Neither step is implicit: `Type(...)` selects the initializer and supplies its arguments; `.new()` allocates and runs it. Writing bare `Type.new()` is a compile-time error (no initializer expression to run), and `Type(...)` without a trailing `.new()` is an incomplete expression and is likewise rejected. The lifecycle verbs `new()`, `free()`, and `freeDeep()` **all take no arguments** — arguments belong exclusively to the initializer expression / `init(...)` body.

Construction may be restricted per class via `@noNew` / `@noInit` (§11.8).

> **Why `Type(args)` then `.new()`, not `.new(args)`:** arguments attach to the initializer — the phase that consumes them — while `new` (allocation) stays argument-free. The reverse chaining (`new().init(...)` or `new(args)`) is deliberately **not** offered: it would either expose the raw, uninitialized allocation as a usable value, or conflate allocation with initialization.

---

### 11.2 Field Initialization Order

Fields are initialized in **declaration order**, top to bottom, as they appear in the class body.

```bestie
class Connection {
    host: str        // initialized first
    port: int        // initialized second
    socket: Socket   // initialized third
}
```

Rules:

* Each field must be assigned exactly once inside `init()` before any use
* A field may not be read before it is assigned in the same `init()` body
* The compiler enforces full initialization of all fields before `init()` returns
* Fields with default values (e.g., `port: int = 80`) are treated as if the assignment appears at the top of `init()` before any user code, in declaration order

---

### 11.3 Base-Class Initialization Order and the Constructor Prologue

In a derived class, every `init()` must call `super.init(...)` **exactly once** on every path. It need not be the literal first statement: it may be preceded by a **prologue** — a run of statements that does not touch the instance under construction.

```bestie
open class Shape {
    color: str

    init(color: str) {
        this.color = color
    }
}

class Circle ext Shape {
    radius: int

    init(color: str, radius: int): ! ShapeError {
        if radius <= 0 { return !BadRadius }   // prologue: validate before building the base
        super.init(color)                      // ends the prologue; base now initialized
        this.radius = radius                   // derived fields follow
    }
}
```

**Prologue rules:**

* A prologue statement may read the `init()` parameters and locals, compute values, and `return !...` from a fallible `init()`.
* A prologue statement may **not** read or write any field of `this`, call any method on `this` or `super`, take `this.address()`, or otherwise observe the instance being constructed. Violations are a compile-time error.
* `super.init(...)` — or a delegating `this.init(...)` (§11.5) — **ends the prologue**. Exactly one such call must be reached on every path through the `init()` body.
* After `super.init(...)` returns, all base fields are fully initialized, `this` becomes accessible, and derived-field assignment proceeds in declaration order (§11.2).

**Rules:**

* No access to `this` fields or methods before `super.init(...)` — only the `this`-free prologue may precede it
* The compiler rejects any derived-class `init()` that does not reach `super.init(...)` (or a delegating `this.init(...)`) exactly once on every path
* All base fields are fully initialized before any derived field assignment begins
* This applies transitively — if the base itself derives from another class, its own `init()` follows the same prologue-then-`super.init(...)` shape
* Since Bestie supports single inheritance only (`ext` binds exactly one class), initialization order is always a linear chain with no diamond ambiguity
* Because the prologue cannot touch `this`, a failure returned from the prologue needs **no field drops** — nothing has been initialized yet. This is strictly cheaper than failing after `super.init(...)`, which must drop the base (§11.6).

> **Why narrower than Java's flexible constructor bodies (JEP 482):** Java's main motivation is initializing subclass fields *before* `super()` so base-class virtual calls don't observe uninitialized derived state. Bestie forbids `@virtual` calls in `init()` entirely (§11.7), so that motivation does not apply — the prologue exists purely for early validation/argument computation and pairs with fallible construction (§11.6). Writing `this` fields before `super.init(...)` remains forbidden, preserving the linear "base fully initialized before any derived field" invariant.

---

### 11.4 Compiler-Generated Memberwise Initializer

If no `init()` is declared, the compiler generates a **memberwise initializer**: one parameter per field, in declaration order, with each parameter name matching the field name.

```bestie
data class Point {
    x: int
    y: int
}
// compiler generates: init(x: int, y: int)
// called as: Point(x: 3, y: 4).new()
```

Rules:

* The compiler-generated init is **removed entirely** once any explicit `init()` is declared. No implicit default-argument constructor is retained.
* For a derived class with no explicit `init()`, the compiler generates a memberwise init that calls `super.init(...)` for the base fields first, then initializes derived fields, in declaration order.
* `data class` and `value class` always receive a compiler-generated memberwise init unless `@noInit` suppresses it.
* If a field has no default value and no `init()` is declared, the field **must** appear as a parameter in the generated init — there is no zero-initialization of arbitrary types.

---

### 11.5 Multiple `init()` Overloads

A class may declare multiple `init()` overloads, distinguished by parameter types (standard overload resolution applies).

```bestie
class Server {
    host: str
    port: int

    init(host: str, port: int) {
        this.host = host
        this.port = port
    }

    init(host: str) {
        this.init(host, port: 80)    // delegating constructor
    }
}
```

Rules:

* An `init()` may delegate to another `init()` overload of the same class using `this.init(...)`. Like `super.init(...)`, it ends the prologue (§11.3) and may be preceded only by `this`-free statements
* `this.init(...)` and `super.init(...)` are mutually exclusive — exactly one applies per `init()` body, and it ends the prologue
* Delegation chains must terminate; circular delegation is a compile-time error
* All paths through every `init()` overload must fully initialize all fields before returning

---

### 11.6 Fallible Construction

`init()` may declare a recoverable failure via the `!` return type.

```bestie
errors ConnectionError { InvalidHost, PortOutOfRange }

class Connection {
    host: str
    port: int

    init(host: str, port: int): ! ConnectionError {
        if port > 65535 { return !PortOutOfRange }
        this.host = host
        this.port = port
    }
}

val conn = Connection("localhost", 9000).new() catch |err| { ... }
```

Rules:

* Fallible `init()` propagates using the standard `!` error-return mechanism
* On error return, the compiler inserts field-drop logic for any fields already initialized, in **reverse initialization order**
* Base-class fields (initialized via `super.init()`) are dropped after all derived fields are dropped — the reverse of the initialization chain
* The partially-constructed object is freed before the error is returned to the caller; the caller never holds a partially-initialized instance
* Panic during `init()` is unrecoverable; the program terminates immediately. Partially-initialized fields are not dropped. This is consistent with the language-wide guarantee that panics do not unwind.

---

### 11.7 Virtual Methods Forbidden During Construction

Calling `@virtual` methods from within any `init()` body is a **compile-time error**.

```bestie
open class Base {
    @virtual fun configure() { ... }

    init() {
        this.configure()    // error: @virtual call forbidden in init()
    }
}
```

Reason: during `init()`, the derived class is not yet fully initialized. Dispatching to an overridden method in the derived class would access uninitialized derived fields.

Rules:

* Any call to a `@virtual`-annotated method within an `init()` body — on `this`, on `super`, or on a field value — is rejected at compile time
* Calls to non-virtual methods and protocol methods with static dispatch are permitted
* This restriction applies to all `init()` overloads, including delegating constructors

---

### 11.8 Construction Restrictions (Annotations)

| Annotation | Effect |
| ------------ | ------- |
| `@noNew` | Prevents the `.new()` materialization verb at external call sites. Forces use of a factory function. The class may still be materialized internally by a factory. |
| `@noInit` | Suppresses the compiler-generated memberwise initializer and forbids `Type(...)` at external call sites. Used for types built entirely via static factory methods or FFI. |
| `@noConstruct` | Combines `@noNew` and `@noInit`. No user-visible construction path exists. Useful for singleton types, opaque handles, and types managed entirely by a runtime or external system. |

```bestie
@noNew
class DbHandle {
    init(conn: RawConn) { ... }     // callable internally by factory only
}

fun openDb(dsn: str): DbHandle ! DbError {
    ...
    return DbHandle(conn).new()    // permitted inside the module
}
// DbHandle(...).new() at an external call site → compile-time error
```

---

### 11.9 Inner Class Construction

Inner classes have no implicit reference to the outer instance (see section 8). Their construction follows the same rules as any top-level class.

```bestie
class Outer {
    class Inner {
        val: int
        init(val: int) { this.val = val }
    }

    inner: Inner

    init() {
        this.inner = Inner(val: 42).new()    // outer explicitly constructs inner
    }
}
```

Rules:

* The outer class's `init()` is responsible for constructing any inner class instances it owns, as part of its normal field initialization order
* An inner class's `init()` cannot implicitly reference the outer instance — any access must be passed explicitly as a parameter
* Inner class construction failure (fallible `init()`) propagates to the enclosing `init()` via the standard `!` mechanism and triggers field drops as per section 11.6
* `value class` inner classes are copy-initialized; their fields are laid out inline in the outer object's memory

---

### 11.10 No Implicit Zero Initialization

Bestie **never zero-initializes fields implicitly**. Every field must receive a value either from a default expression in the class body or from every code path through every `init()`. Forgetting to initialize any field is a **compile-time error**.

```bestie
class Counter {
    count: int          // ❌ no default, no init() — compile-time error
}

class Counter {
    count: int = 0      // ✅ explicit default in class body
}
```

This rule applies to all class kinds: `class`, `open class`, `abstract class`, `data class`, and `value class`.

There is no "zero state" implicitly assigned to any type. Absence of a value is not the same as zero — if a field genuinely may not hold a value, declare it as `T ?`:

```bestie
import bestie.lib.utilities.option

class Session {
    userId: int
    token: str ?                 // explicitly absent until authenticated

    init(userId: int) {
        this.userId = userId
        this.token = option.Not_Present
    }

    fun authenticate(tok: str) {
        this.token = option.Present(tok)
    }
}
```

Using `T ?` for lazy or conditional fields makes the absent state a first-class part of the type. The caller must handle both present and absent — no accidental null dereference, no silent missing-value bug. Named constructors come from std-lib; `if-let` on `token` needs no import.

---

### 11.11 `@Initialize` — Third-Party Plugin Convention

The core language enforces explicit initialization for all fields. For use cases where zero or default initialization of an entire class body is desirable — for example, configuration holders or data-transfer objects — a **third-party compiler plugin** may provide an `@Initialize` annotation that generates default field values automatically:

```bestie
// Provided by a third-party plugin — not part of the core language
@Initialize
class Config {
    maxConnections: int     // plugin generates: = 0
    timeout: float64        // plugin generates: = 0.0
    host: str               // plugin generates: = ""
}
```

The plugin synthesizes the appropriate zero/default value for each field based on its type, removing the need to write `= 0` or `= ""` everywhere when that is the desired behavior.

The core language itself is unchanged: without the plugin, `@Initialize` is an unknown annotation and the compiler will still reject uninitialized fields. This is explicitly an **opt-in escape hatch**, in the spirit of Java's Lombok project — extending the language through the plugin system without compromising the core's explicit-initialization guarantee.

---

### 11.12 Destruction (`deinit`)

Bestie has **no C++-style destructor and no RAII**. `deinit()` is never invoked implicitly and never runs at scope exit. It is an explicit, deterministic cleanup hook that runs **only** because the programmer calls `free()` or `freeDeep()`. See `core/memory.md` §7 for how the call site drives it and why this does not reintroduce implicit cleanup.

A class may declare at most one `deinit()`, for teardown that field drops alone cannot express — closing a socket, releasing an FFI handle, or `c.free()`-ing a `ptr<T>` field (see `memory.md` §10.1.1):

```bestie
class Connection {
    socket: ptr<RawSocket>

    init(host: str): ! ConnectionError {
        this.socket = os.openSocket(host) catch |e| { return !Unreachable }
    }

    deinit() {
        os.closeSocket(this.socket)    // teardown co-located with the type
    }
}

val own c = Connection("localhost").new() catch |e| { ... }
c.free()    // runs c.deinit(), then releases c's storage
```

**Rules:**

* `deinit()` takes no parameters and declares no return type. It **cannot** be fallible (no `! ErrorSet`); an unrecoverable cleanup problem must `panic`.
* `deinit()` runs when `free()` or `freeDeep()` is explicitly called on a live, fully-constructed instance. It is **not** called at scope exit, and the compiler still requires explicit discharge of every ownership obligation (`memory.md` §7.4).
* **Execution order** — `x.freeDeep()`: (1) run `x.deinit()`; (2) drop `own` fields in **reverse** declaration order; (3) release `x`'s own storage. `x.free()`: run `x.deinit()`, then release only `x`'s storage — its direct `own` fields must already have been discharged (`memory.md` §7.1).
* **Hierarchy chaining** — for an `open`/`abstract class` hierarchy, `deinit()` chains **most-derived first**, then each base `deinit()` up the chain, reversing the `super.init(...)` order. The compiler inserts the base calls automatically; do **not** call `super.deinit()` manually.
* `deinit()` may read fields and call non-`@virtual` methods. Calling a `@virtual` method from `deinit()` is a compile-time error, for the same reason as §11.7: while a base subobject is being destroyed, the derived part may already be torn down.
* `deinit()` must not move `this`, transfer ownership of `this`, or create a new owning alias of the instance — **resurrection is forbidden**.
* `deinit()` is **not** invoked on construction failure. If a fallible `init()` returns an error, only the already-initialized fields are dropped in reverse order (§11.6); a never-completed object has no `deinit()`. `deinit()` pairs only with a *completed* `init()`.
* A `panic` inside `deinit()` terminates the program immediately, consistent with the language-wide no-unwind guarantee. Fields not yet dropped are not cleaned up.
* Only reference class kinds may declare `deinit()`: `class`, `open class`, `abstract class`. `value class`, `data class`, and `enum` cannot — they hold no `own`/`ref` fields (`memory.md` §10.1.1) and thus own no resource requiring teardown. An `enum` with an `own` payload still has its owned field freed by `freeDeep()`, but hosts no `deinit()` body.

---

## 12. Sealed Declarations (File-Scoped)

Sealing defines **closed, finite sets** of declarations.

Applies to:

* Classes
* Protocols
* File-level functions

No reflection or registration overhead.
Sealing may improve dispatch and layout because the full implementor set is known at compile time.

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
* Sealing may replace pointer-sized runtime metadata with compact type tags
* Calls through sealed hierarchies may lower to direct tag-switch dispatch instead of vtables

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
* Singleton-style global objects — use atomics or locks from `bestie.lib.concurrency`

---

## 14. What Bestie Deliberately Avoids

* Implicit virtual methods
* Multiple inheritance
* General-purpose runtime RTTI APIs
* Reflection-based dispatch
* Fragile base classes
* Language-level singleton types
* Anonymous classes / inline object expressions (use a named class or a lambda — see section 4.3)

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

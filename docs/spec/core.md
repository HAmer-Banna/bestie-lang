# Bestie Core Language Specification

This document defines the **Bestie core language**.

Everything described here is:

* **First-class**
* **Always available**
* **Stable by design**
* **Independent of libraries and frameworks**

The core language is intentionally **explicit**, **performance-oriented**, and **semantically closed**.
All higher-level capabilities are layered through `std-lib`, `std-api`, and `std-frameworks`.

---

## 1. Core Design Principles

The Bestie core is governed by the following non-negotiable principles:

1. **Native speed is mandatory**
2. **Compile-time resolution whenever possible**
3. **Explicit memory and ownership**
4. **No garbage collection**
5. **No hidden allocation**
6. **No unsafe escape hatches**
7. **Unified model for system and backend**
8. **Convention over configuration**
9. **Professional clarity over cleverness**

If a feature exists in the core, it is intended to be used.

---

## 2. Naming Conventions

### 2.1 Keywords and reserved identifiers

* All keywords are **lowercase**
* Built-in types and primitives are **lowercase**

Examples:

```
int32, int64, float32, bool, str, list, map, threadOS
```

---

### 2.2 User-defined symbols

* Classes, enums, protocols: **PascalCase**
* Functions, variables, properties: **camelCase**

Java/Kotlin conventions are followed intentionally.

---

## 3. Core Types

### 3.1 Primitive value types (inlined)

```
int32
int64
uint
float32
float64
byte
bool
char
str
void
```

Properties:

* Value types
* Inlined
* Passed by value unless explicitly referenced

`void` is used for functions or methods that return no value.

---

### 3.2 Compound types

```
tuple
ptr<T>
function types
```

Tuples:

* Are value types
* Support destructuring
* Are stack-allocated when possible

---

## 4. Classes and Type Declarations

### 4.1 Class kinds

Bestie supports:

```
data class
value class
single class
abstract class
class
open class
enum
enum class
```

Rules:

* All classes are **inlined by default**
* `open class` may be non-inlined if required
* Single inheritance only
* Multiple protocol implementation allowed

---

### 4.2 Inheritance and protocols

```
class A ext B
class A impl P1, P2
class A impl GroupProtocol
```

Rules:

* No multiple inheritance
* No MRO
* Conflicts resolved explicitly (Java 8 style)

---

## 5. Object Lifecycle

### 5.1 Initialization vs Allocation

Bestie separates:

* **Initialization** (value setup)
* **Allocation** (memory reservation)

There are no implicit constructors.

---

### 5.2 Initialization (`init`)

Initialization functions:

* Are explicit
* Return `Self`
* Do not allocate memory

Example:

```
class User {
    name: str
    age: int

    fun init(name: str, age: int): User {
        this.name = name
        this.age = age
        return this
    }
}
```

Rules:

* `init` may be overloaded
* Compiler enforces full initialization
* May be `priv`, `pkg`, `pub`

---

### 5.3 Allocation (`new()`)

Allocation is explicit:

```
own u = User.new()
```

* Produces `own T`
* Does not initialize fields
* Uninitialized access is a compile-time error

---

### 5.4 Combined pattern (canonical)

```
own u = User.init(name = "Ali", age = 20).new()
```

Evaluation order:

1. Initialize value
2. Allocate memory
3. Transfer ownership

---

## 6. `this` and `super`

### 6.1 `this`

* Refers to the current instance
* Non-nullable
* Explicit only

---

### 6.2 `super`

* Explicit access to parent implementation
* Required for parent initialization

Example:

```
class User ext Person {
    fun init(name: str, age: int): User {
        super.init(name)
        this.age = age
        return this
    }
}
```

No implicit constructor chaining exists.

---

## 7. Properties

### 7.1 Basic properties

```
class Student {
    name: str { get, set }
}
```

Compiled as methods:

* `getName(): str`
* `setName(str): void`

---

### 7.2 Custom accessors

```
class Student {
    name: str {
        get() { return first + " " + last }
        set
    }
}
```

Rules:

* Properties obey ownership rules
* Properties resolved at compile time
* No properties in protocols

---

### 7.3 Property overriding

Allowed only if:

* Property is `open`
* Type is identical
* Ownership qualifier is identical

Requires `@override`.

---

## 8. Inner Classes and Functions

### 8.1 Inner classes

Inner classes:

* Are full classes
* Obey visibility modifiers
* Are accessed via `Outer.Inner`

Visibility:

* Defaults to `priv` relative to outer class
* May be declared `pub` or `pkg`

No implicit binding to outer instance.

---

### 8.2 Inner functions

Inner (local) functions:

* Lexically scoped
* Compile-time resolved
* Cannot escape their scope

Used for clarity and optimization.

---

## 9. Functions and Methods

### 9.1 Function features

Supported:

* Inline functions
* Concise bodies
* Default parameters
* Named arguments
* Local functions
* Multiple return values (tuples)

Example:

```
fun sum(a: int, b: int = 0): int
```

---

### 9.2 Overloading

* **Method overloading**: allowed
* **Function overloading**: allowed with restrictions

Rules:

* Overloads must differ in arity
* No ambiguous signatures
* No implicit conversions

---

## 10. Data Structures (Core Language)

Data structures are **first-class language constructs**, not libraries.

```
list<T>
set<T>
map<K, V>
deque<T>
heap<T>
```

Variants are compile-time selectable:

```
list.asArray
list.asMatrix
set.asHash
map.asTree
deque.asQueue
```

Literals:

```
val l: list<int> = {1, 2, 3}
```

---

## 11. Memory and Ownership

Core constructs:

```
own
ref
ptr<T>
address()
deref()
free()
freeDeep()
```

Rules:

* Pass-by-value by default
* Ownership is explicit
* No null
* No use-after-free
* Compiler enforces lifetimes

(Full details in `memory-layout.md`)

---

## 12. Error Handling (Core)

Bestie uses **typed error unions**.

```
fun read(): File | IOError
```

Rules:

* No exceptions
* Errors must be handled or propagated
* Compile-time enforced

---

## 13. Concurrency (Core)

Concurrency primitives are part of the core language:

```
threadOS
threadLight
channel<T>
mutex<T>
atomic<T>
```

Rules:

* Ownership-aware
* No data races
* Deterministic by construction

(Extended models live in `std-api.extended-concurrency`)

---

## 14. Generics

* Compile-time only
* No erasure
* No variance keywords
* Explicit instantiation

---

## 15. Modules and Visibility

Visibility modifiers:

```
pub
pkg
protec
priv
```

Rules:

* `pub` and `protec` require export via `bestie.mod`
* Package boundaries are strict
* No reflective access

---

## 16. Strings and Templates

* Double quotes (`"`) support interpolation
* Single quotes (`'`) are literal only

```
"Hello ${user.name}"
'Hello ${user.name}'
```

---

## 17. Compile-Time Annotations

Annotations:

* Are compile-time only
* Cannot execute code
* Cannot alter semantics implicitly

Predefined annotations:

```
@override
@inline
@noinline
@deprecated
@experimental
@pure
```

No runtime reflection exists in the core language.

---

## 18. Explicit Exclusions

The core language does **not** include:

* Garbage collection
* Reflection
* Dynamic typing
* Unsafe blocks
* Foreign code integration
* Dependency injection
* Framework abstractions

Foreign interaction is handled **exclusively** by `std-api.foreign`.

---

## 19. Stability Guarantee

The core language:

* Changes rarely
* Requires major version bumps
* Preserves source compatibility whenever possible

The core is designed to be **rock-solid and boring** — by intention.


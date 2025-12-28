# Bestie Core Language Specification

This document defines the **Bestie core language**.
Everything described here is **first-class**, **always available**, and **guaranteed stable** across Bestie versions.

The core language is intentionally small, explicit, and performance-oriented.
Higher-level capabilities are layered through `std-lib`, `std-api`, and `std-frameworks`.

---

## 1. Design Principles

The Bestie core is governed by the following principles:

1. **Native performance is non-negotiable**
2. **Compile-time resolution is preferred whenever possible**
3. **Safety without unsafe escape hatches**
4. **Explicit ownership and memory behavior**
5. **OOP, functional, and low-level constructs coexist**
6. **Convention over configuration**
7. **No feature that is “better avoided”**

If a feature exists in the core, it is expected to be used.

---

## 2. Naming Conventions

### 2.1 Keywords and reserved identifiers

* All keywords are **lowercase**
* Built-in types and primitives are **lowercase**

Examples:

```
int32, int64, float32, bool, str, list, map, threadOS
```

### 2.2 User-defined symbols

* Classes, protocols, enums: **PascalCase**
* Functions, variables, properties: **camelCase**

This follows Java/Kotlin conventions intentionally.

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

All primitives are:

* Value types
* Inlined
* Passed by value unless explicitly addressed

`void` is used for functions and methods that return no value.

---

### 3.2 Compound types

```
tuple
ptr<T>
function types
```

Tuples are value types and support destructuring.

---

## 4. Classes and Type Declarations

### 4.1 Class kinds

Bestie supports the following class declarations:

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
* Multiple protocol implementation is allowed

---

### 4.2 Inheritance and protocols

```
class A ext B
class A impl P1, P2
class A impl GroupProtocol
```

* No multiple inheritance
* No method resolution order (MRO)
* Conflicts are resolved explicitly, similar to Java 8

---

## 5. Properties

Properties provide structured state access and are compiled into methods.

### 5.1 Basic property

```
class Student {
    name: str { get, set }
}
```

Compiles into:

* `getName(): str`
* `setName(str): void`

---

### 5.2 Custom accessors

```
class Student {
    name: str {
        get() {
            return name.firstName + " " + name.lastName
        }
        set
    }
}
```

Rules:

* Properties participate in ownership rules
* Properties may return `ref`, `own`, or value types
* Properties are resolved at compile time
* No properties in protocols

---

## 6. Constants

Bestie does not support `static`.

Constants are declared at the file or package level:

```
math.PI
math.E
```

Constants are:

* Compile-time resolved
* Immutable
* Namespaced by package

---

## 7. Functions

### 7.1 Function forms

```
fun f(): int
fun f(x: int): void
fun f(x: int) = x + 1
```

Supported features:

* Inline functions
* Concise bodies
* Default parameters
* Local functions
* Multiple return values via tuples
* `_` for unused parameters

---

### 7.2 Lambdas

Lambdas are resolved at compile time unless explicitly required otherwise.

```
val inc = { x: int -> x + 1 }
```

---

## 8. Control Flow

All control structures are both **statements and expressions**.

```
if / else
switch (with pattern matching)
while
for
for (in)
```

Examples:

```
val x = if a > b a else b
val sum = for i = 0; i < 10; i++ i
```

All branches must be exhaustive.

---

## 9. Data Structures (First-Class Core Types)

Data structures are part of the **core language**, not libraries.

### 9.1 Collections

```
list<T>
set<T>
map<K, V>
deque<T>
heap<T>
```

### 9.2 Representations

Each structure supports compile-time variants:

```
list.asArray
list.asMatrix
list.asLinked

set.asHash
set.asTree
set.asLinked

map.asHash
map.asTree
map.asLinked

deque.asStack
deque.asQueue

heap.asMin
heap.asMax
```

Defaults exist for all except `heap`.

### 9.3 Literals and builders

```
val l: list<int> = {1, 2, 3}
val m = map<int, str>.of(1, "a", 2, "b")
```

Builders and factories are enforced via protocols.

---

## 10. Memory Model

### 10.1 Core concepts

```
ptr<T>
ref
own
address()
deref()
free()
freeDeep()
```

Rules:

* Pass-by-value by default
* References are explicit
* Ownership is explicit
* No null values
* Compiler enforces lifetime correctness

---

## 11. Error Handling

Bestie uses **typed error unions**, inspired by Zig.

```
fun readFile(path: str): File | IOError
```

Handling:

```
val f = try readFile("a.txt")
val f = readFile("a.txt") catch defaultFile
```

Rules:

* No exceptions
* No unchecked errors
* Errors must be handled or propagated
* Fully compile-time enforced

---

## 12. Concurrency (Core Primitives)

Concurrency is a **first-class language feature**.

### 12.1 Core primitives

```
threadOS
threadLight
```

* OS threads and lightweight threads
* CSP-style communication
* Compile-time resolved where possible

Advanced models live in `std-api`.

---

## 13. Generics

* Compile-time only
* No type erasure
* No `extends` / `super`
* Explicit and predictable instantiation

---

## 14. Modules and Visibility

Visibility modifiers:

```
pub
pkg
protec
priv
```

Rules:

* `pub` and `protec` require export via `bestie.mod`
* Package boundaries are strictly enforced
* Modules are designed for long-term stability

---

## 15. String Literals and Templates

* Double quotes (`"`) support interpolation
* Single quotes (`'`) are literal-only

```
val s = "Hello ${user.name}"
val t = 'Hello ${user.name}'
```

---

## 16. Compile-Time Annotations

Annotations are resolved at compile time only.

Example:

```
@override
```

No runtime reflection exists in the core language.

---

## 17. What the Core Explicitly Excludes

The core language does **not** include:

* Garbage collection
* Reflection
* Unsafe blocks
* Dynamic typing
* Runtime metaprogramming
* Dependency injection
* Framework-level abstractions

These are layered above the core.

---

## 18. Stability Guarantee

Changes to the core language:

* Are rare
* Require major version increments
* Preserve source compatibility whenever possible

The core is designed to be **solid as a rock**.

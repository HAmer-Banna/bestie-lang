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

9.3 Unified Callable Model

Bestie uses one keyword for all callable constructs:

fun

There is:

No alternate lambda keyword

No symbolic-only function definitions

This unifies:

Functions

Methods

Lambdas

Function types

9.4 Function Definitions
Standard form
fun add(x: int, y: int): int {
    return x + y
}
Concise body (expression function)
fun add(x: int, y: int) = x + y

Rules:

return may be omitted when the body is a single expression

Return type may be inferred

Semicolons follow Kotlin rules:

Optional at line end

Required only when multiple statements share one line

Concise bodies are encouraged for clarity and readability.

9.5 Function Types

Function types explicitly use fun:

fun(int, int): int
fun(User): void

Example:

val op: fun(int, int): int = add

This avoids symbolic ambiguity and improves tooling clarity.

9.6 Lambdas (Fat Arrow Syntax)

Lambdas are differentiated from concise function bodies using the fat arrow (=>).

Anonymous lambda
val sum = (x: int, y: int) => x + y
Typed lambda
val sum: fun(int, int): int = (x, y) => x + y
Multi-line lambda
val check = (age: int) => {
    if age < 10 {
        return error
    }
    return age
}

Rules:

Lambdas cannot capture outer variables

Only parameters and constants are allowed

Multi-line lambdas must use braces and return

Lambdas are inlined by default

Zero allocation

Compile-time resolved

Illegal (capture):

val x = 10
val f = (y: int) => x + y   // compile-time error
9.5 Method References

Method references are supported when signatures match exactly:

User::validate
Math::sqrt

Rules:

No partial application

No implicit this

No capture

Fully resolved at compile time

9.7 Currying

Currying is not implicit.

Because closures are disallowed, curried functions must be expressed explicitly and statelessly.

This prevents:

Hidden allocations

Lifetime complexity

Implicit captures

Bestie favors explicit data flow over functional cleverness.

9.8 Extensible (Extension) Functions

Bestie supports extensible functions as compile-time sugar.

Example:

fun list<int>.sum(): int {
    var total = 0
    for v in this {
        total += v
    }
    return total
}

Rules:

Resolved statically

Do not modify the original type

Respect package visibility

Cannot access private members

Must be imported explicitly if defined externally

Extensible functions are encouraged for readability and domain modeling.

9.9 Inlining Rules
@inline

Compile-time annotation

Requests mandatory inlining

If the compiler determines:

The function is too large

Inlining breaks guarantees

➡ Compilation fails

Automatic inlining

Compiler may inline any suitable function

Even without @inline

Based on size, call-site clarity, and ownership safety

Inlining is treated as a correctness feature, not only an optimization.

9.10 Abstract Functions
abstract fun compute(): int

Rules:

Allowed only in abstract class or protocol

No body permitted

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

## 13. Concurrency (Core Language)

Concurrency in Bestie is a **core language feature**, but the core provides only the **minimal primitives required to express parallel execution safely**.

The core language does **not** provide:

* Locks
* Atomics
* Shared-memory synchronization
* Schedulers
* Executors
* Async/await
* Futures or promises

All advanced concurrency abstractions are defined in **`std-api.extended-concurrency`**.

---

## 13.1 Core Concurrency Primitives

The core language defines exactly **two execution primitives**:

```
threadOS
threadLight
```

These are **language-level constructs**, not library types.

---

### 13.1.1 `threadOS`

`threadOS` represents a **native operating system thread**.

Characteristics:

* Maps 1:1 to an OS thread
* Has its own stack
* Is scheduled by the operating system
* Has a stable identity

Usage:

```
threadOS.spawn(fn)
```

Rules:

* `threadOS` boundaries are **ownership boundaries**
* Values passed into a `threadOS` must obey ownership transfer rules
* No implicit sharing is allowed

---

### 13.1.2 `threadLight`

`threadLight` represents a **lightweight user-managed thread**.

Characteristics:

* Scheduled by the Bestie runtime
* May multiplex over one or more `threadOS`
* Has explicit yield points
* No preemption guarantees

Usage:

```
threadLight.spawn(fn)
```

Rules:

* `threadLight` execution is deterministic when possible
* Blocking OS calls are forbidden inside `threadLight`
* Ownership rules are enforced identically to `threadOS`

---

## 13.2 Ownership as the Concurrency Model

Bestie uses **ownership**, not synchronization, as its primary concurrency safety mechanism.

### 13.2.1 Ownership Transfer

Rules:

* `own T` may be **moved** into a thread
* After move, the original binding becomes invalid
* Exactly one owner exists at any time

Example:

```
own data = Data.init().new()

threadOS.spawn(fn () {
    consume(data)
})
```

This is safe by construction.

---

### 13.2.2 Borrowing (`ref`)

Rules:

* `ref T` cannot cross thread boundaries
* Borrowed references are **thread-local**
* The compiler rejects cross-thread `ref` usage

Example (compile-time error):

```
ref x = data
threadOS.spawn(fn () {
    use(x)   // ❌ illegal
})
```

---

## 13.3 Functions and Lambdas in Concurrency

### 13.3.1 Function Passing Rules

Functions passed to threads must:

* Have fully resolved types
* Capture values explicitly
* Obey ownership rules

There is no implicit capture.

---

### 13.3.2 Lambdas

Lambdas are **explicit capture closures**.

Example:

```
own v = Value.init().new()

threadOS.spawn(fn [v] () {
    process(v)
})
```

Rules:

* Captured `own` values are moved
* Captured `ref` values are forbidden
* Captures are explicit and visible

---

## 13.4 Memory Visibility

Memory visibility rules are simple:

* Ownership transfer establishes a happens-before boundary
* No shared mutable memory exists in the core model
* Data races are impossible by construction

If shared state is required, it must be expressed through **higher-level APIs**.

---

## 13.5 What the Core Deliberately Excludes

The core language does **not** include:

* Mutexes
* Spinlocks
* Atomics
* Condition variables
* Channels
* Message queues
* Thread pools
* Async runtimes
* Structured concurrency

These belong exclusively to:

```
std-api.extended-concurrency
```

---

## 13.6 Rationale

This design ensures that:

* Concurrency safety is **compile-time enforced**
* There are **no data races**
* There is **no undefined behavior**
* The core remains small and analyzable
* Advanced concurrency remains opt-in

The core language provides **execution**, not coordination.

---

## 13.7 Summary

Core concurrency in Bestie consists of:

* `threadOS`
* `threadLight`
* Ownership transfer
* Explicit function and lambda execution

Everything else is a **library-level abstraction**, not a language concern.

---

## 14 Generics

14.1 Overview

Bestie generics are:

Compile-time only

Fully monomorphized

No type erasure

Zero runtime overhead

There are no variance keywords.

14.2 Generic Class Example
class Box<T> {
    value: T

    fun init(value: T): Box<T> {
        this.value = value
        return this
    }
}

Usage:

own b = Box<int>.init(10).new()
14.3 Generic Function Example
fun identity<T>(value: T): T = value
14.4 Generic Enum Class
enum class Result<T> {
    Ok(T),
    Err(str)
}
14.5 Why No Variance Keywords

Variance:

Increases cognitive load

Encourages over-generalization

Introduces runtime assumptions

Bestie relies instead on:

Ownership

Explicit movement

Value semantics

This yields simpler, safer generics.

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

17.3 Custom Compile-Time Annotations

Definition
annotation ValidateRange(min: int, max: int)
Usage
@ValidateRange(min = 0, max = 100)
fun setScore(score: int): void

Rules:

Annotation arguments must be compile-time constants

Enforcement is handled by the compiler or tooling

No runtime presence

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


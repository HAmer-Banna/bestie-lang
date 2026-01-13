# Bestie Core Language Specification

Everything described here is **first-class, always available, stable, and sealed**.  
Higher-level capabilities are in `std-lib`, `std-api`, and `std-frameworks`.

---

## 1. Core Principles

1. Native speed mandatory
2. Compile-time resolution
3. Explicit memory & ownership
4. No garbage collection
5. No hidden allocation
6. No unsafe escape hatches
7. Unified system + backend model
8. Professional clarity over cleverness

---

## 2. Naming Conventions

* Keywords: lowercase
* Types: lowercase
* Classes/enums/protocols: PascalCase
* Functions/variables/properties: camelCase

---

## 3. Types

### 3.1 Primitive Types

`int32, int64, uint, float32, float64, byte, bool, char, str, void`

### 3.2 Compound Types

`tuple, ptr<T>, function types`

* Tuples are value types, stack-allocated when possible.

---

## 4. Classes & Protocols

### 4.1 Class Kinds

`data class, value class, single class, abstract class, open class, enum, enum class`

* All classes inline by default
* Single inheritance only
* Multiple protocols allowed
* Sealed classes supported
* `@immutable` enforces immutability on close or abstract derived classes
* Compiler warning if `@immutable` used on already immutable data/value/enum

---

## 5. Memory & Ownership

Core constructs:

`own, ref, ptr<T>, address(), deref(), offset(), free(), freeDeep()`

### Rules

1. `own` = sole owner, can move
2. `ref` = borrowed, thread-local, cannot cross threads
3. `ptr<T>` = low-level pointer, only necessary when direct memory access required
4. `address()` returns address of any object or function
5. Copy vs deepCopy behavior:
   - Copy duplicates the value, deepCopy duplicates objects recursively
   - Address of object may or may not change depending on allocation

---

## 6. Functions & Return Types

### 6.1 Function Declaration

```bestie
fun name(params): T { ... }
val lambda: fun(int, int): int = (x, y) => x + y
Lambdas use ()=>{} syntax

No capture of outer variables

Fully resolved at compile-time

Method references supported: Class::method

6.2 Return Types
Complete: fun getUser(): User → always returns a value

Partial: fun getUser()?: User → caller must handle with if/switch

Option: Option<User> → two states: Present or Not_Present

Error: fun read(): File!IOError → Zig-style error union

Compiler enforcement:

Partial functions (T?!) must be handled at call site

Option returns are explicit in std-lib

Compiler shows error if a partial function is called without handling

7. Annotations & Plugins
Predefined: @inline, @noinline, @override, @pure, @deprecated, @experimental

Custom plugins:

bestie
Copy code
annotation ValidateRange(min: int, max: int)

@ValidateRange(min=0, max=100)
fun setScore(score: int)
Compiler or plugin enforces semantics

Compile-time only

Cannot execute code directly

Example: @get, @post in std-framework.web for REST routing

8. Concurrency Primitives
threadOS → OS threads

threadLight → lightweight user threads

Ownership enforces thread safety

Ref cannot cross threads

Own may be moved into threads

No shared mutable state in core

9. Generics
Fully monomorphized at compile-time

No type erasure

Ownership + movement ensures safety

Zero runtime overhead

10. Error Handling
Typed error unions

No exceptions

Errors must be handled or propagated

11. Control Structures
if(...) {} / for(...) {} / while(...) {} → parentheses required

Switch: no fall-through

Operators: support both symbolic (&&) and word-based (and)

12. Stability & Exclusions
Core does not include:

Garbage collection

Reflection

Dynamic typing

Unsafe blocks

Global mutable vars (only val allowed)

Foreign code integration

Framework abstractions

All advanced features exist in std-api or std-frameworks.

Functional Programming (FP) in Bestie

This document defines the functional programming model of the Bestie language. FP in Bestie is explicit, statically analyzable, and compile-time enforced, with no reliance on runtime magic, nullability, or implicit control flow.

Bestie FP is designed to:
	•	Be predictable at compile time
	•	Compose naturally with OOP and systems programming
	•	Avoid hidden allocations and runtime penalties
	•	Eliminate ambiguity between “no value” and “not returned”

⸻

1. Functions

1.1 Function Declaration

Functions are declared using the fun keyword.

fun add(a: int, b: int): int {
    return a + b
}

Rules:
	•	fun is the only function keyword
	•	Return types are explicit (except for void)
	•	There is no function-level type inference for return types

⸻

1.2 Void Functions

Functions that do not return a value use void:

fun log(msg: str): void {
    println(msg)
}

Notes:
	•	return; is allowed but discouraged
	•	The compiler emits a warning if return; is used in void

⸻

2. Complete vs Partial Functions

Bestie replaces null, nil, undefined, and sentinel values with a compile-time distinction between complete and partial functions.

There is no runtime empty value.

⸻

2.1 Complete Functions

A complete function guarantees that it returns a value on all execution paths.

fun getUser(id: int): User {
    return repository.find(id)
}

Rules:
	•	All control-flow paths must return a value
	•	The compiler proves completeness
	•	Consumers do not need guards

Invalid:

fun f(): int {
    if (cond) {
        return 1
    }
}

Compile-time error: not all paths return a value.

⸻

2.2 Partial Functions

A partial function explicitly declares that it may not return a value.

Syntax:

fun getUser(id: int): User ?

With body:

fun getUser(id: int): User ? {
    if (exists(id)) {
        return repository.find(id)
    }
    return
}

Rules:
	•	? marks partiality
	•	return; is only valid in partial or void functions
	•	No runtime representation of absence exists

⸻

2.3 Calling Partial Functions

Calling a partial function forces the caller to handle control flow explicitly.

fun process(id: int): void {
    val user = getUser(id)
    if (user ?) {
        sendEmail(user)
    }
}

The compiler enforces:
	•	Partial calls cannot be used as complete expressions
	•	Results must be guarded or transformed

⸻

2.4 Lambdas and Partiality

Lambdas may also be complete or partial.

Complete lambda:

val inc = (x: int) => x + 1

Partial lambda:

val find = (x: int) => User ? {
    if (x > 0) return repo.get(x)
    return
}


⸻

3. Lambdas

3.1 Syntax

Lambdas use the fat arrow =>.

val sum = (a: int, b: int) => a + b

Block form:

val f = (x: int) => {
    val y = x * 2
    return y + 1
}


⸻

3.2 Lambda Rules
	•	Lambdas are values
	•	Capture is explicit and immutable
	•	No implicit heap allocation

⸻

4. Higher-Order Functions

Functions can accept and return functions.

fun apply(x: int, f: (int) => int): int {
    return f(x)
}

Partial higher-order example:

fun tryApply(x: int, f: (int) => int ?): int ? {
    return f(x)
}


⸻

5. Function Overloading

Bestie supports compile-time overloading.

fun print(x: int): void
fun print(x: str): void

Rules:
	•	Resolution is static
	•	No runtime dispatch
	•	No implicit coercion

⸻

6. Methods vs Functions

Methods are functions bound to a type.

fun User.fullName(): str {
    return first + " " + last
}

Rules:
	•	Same rules as functions
	•	Can be complete or partial

⸻

7. Purity and Side Effects

Bestie does not enforce purity, but encourages it.

Guidelines:
	•	Prefer complete functions
	•	Isolate side effects
	•	Use partial functions to express absence, not failure

⸻

8. Relationship to std-lib FP Utilities

Higher-level FP utilities (map, filter, fold, etc.) live in:

std-lib.functional

The FP core defines:
	•	Function semantics
	•	Lambda behavior
	•	Partiality

The standard library provides combinators, not language magic.

⸻

9. Design Principles
	•	No null
	•	No runtime emptiness
	•	No implicit control flow
	•	Everything is proven at compile time

Bestie FP is intentionally small, explicit, and predictable.
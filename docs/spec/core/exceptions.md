Bestie Exception Handling (exceptions.md)

This document describes how Bestie handles errors and exceptions.

Bestie is designed to be safe, explicit, and predictable. Unlike Java or Python, it avoids:
	•	Runtime surprises
	•	Unchecked exceptions
	•	Hidden null dereferences

⸻

1. Design Goals

Bestie exception handling aims to:
	1.	Be explicit
Users must decide how to handle errors.
	2.	Be type-safe
All error types are strongly typed.
	3.	Avoid hidden runtime costs
Exceptions do not allocate memory implicitly.
	4.	Integrate with core return rules
Works seamlessly with complete, partial, ?!, and option returns.

⸻

2. Exception Types

Bestie distinguishes two primary categories:

2.1 System Errors
	•	Represent runtime errors that usually cannot be recovered from safely.
	•	Examples:
	•	DivideByZeroError
	•	MemoryAllocationError
	•	StackOverflowError
	•	Thrown automatically by the runtime.

fun divide(a: int, b: int): int {
  if b == 0 {
    throw DivideByZeroError("division by zero")
  }
  return a / b
}

2.2 Application Errors
	•	Represent user-defined failures.
	•	Users can define custom error types as classes:

class InvalidUserInputError {
  val message: str
  fun new(msg: str) {
    message = msg
  }
}

	•	Thrown explicitly by the application code.

⸻

3. Throwing Exceptions
	•	Use throw to raise an exception.
	•	Can throw any object that inherits from Error class.
	•	throw is type-checked at compile time.

throw InvalidUserInputError.new("Input must be positive")


⸻

4. Handling Exceptions
	•	Use try/catch blocks to handle exceptions.
	•	Catch blocks are ordered, type-checked, and compile-time verified.

try {
  val x = divide(10, 0)
} catch DivideByZeroError as e {
  print("Cannot divide by zero")
} catch Error as e {
  print("General error: ", e.message)
}

	•	catch can match specific error types or general errors.

⸻

5. Finally Block
	•	Optional finally block executes regardless of exception.

try {
  val f = file.open("config.txt")
} catch FileNotFoundError as e {
  print("File missing")
} finally {
  cleanup()
}


⸻

6. Integration with Core Return Types

Bestie exceptions interact cleanly with the four core return types:

Return Type	Behavior with Exceptions
Complete function	Can throw, caller may catch or propagate
Partial function (?)	Must handle exceptions, compiler ensures proper use
Option class	Exceptions can coexist with Not_Present state
Error return (!)	Designed for recoverable errors, alternative to throw

Bestie encourages using error return (!) for normal recoverable failures, and throw for unexpected runtime failures.

⸻

7. Rules & Guidelines
	1.	Exceptions do not cross thread boundaries implicitly.
	2.	Only Error-derived classes may be thrown.
	3.	throw is explicit, and compiler verifies all paths where a function may throw.
	4.	Runtime exceptions are rare; prefer partial functions, ?!, or option for expected failures.
	5.	Exceptions cannot be silently ignored; compiler warns if not handled.

⸻

8. Best Practices
	•	Prefer error returns for expected failures.
	•	Use throw for unrecoverable or system-level errors.
	•	Combine exceptions with partial functions for maximum clarity.
	•	Avoid mixing throw and option without reason; choose one pattern per function.

⸻

9. Summary
	•	Bestie exception handling is explicit, predictable, and safe.
	•	It complements return types and memory safety rules.
	•	Exceptions are a tool for rare, unrecoverable situations, not general error flow.
	•	Compiler enforces rules at compile time, preventing accidental runtime surprises.

⸻

This design ensures Bestie remains safe for system-level programming, yet expressive enough for high-level backend logic.

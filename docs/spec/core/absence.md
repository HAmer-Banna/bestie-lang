# Absence — `T ?`

This document is the single home of **absence** in Bestie: the `T ?` type, the `absent` value, and the operators that work with them. Other documents cite it.

Bestie has no `null`, `nil`, `NULL`, or `nullptr` (`fp.md` §4). A value that may be missing has a type that says so, and the compiler makes every use of it handle the missing case.

---

## 1. The `?` Family

Everything about absence is spelled with `?`, and everything about failure with `!`:

| | Absence | Failure |
| - | ------- | ------- |
| Type | `T ?` | `T ! E` |
| Produce the empty/failed case | `absent`, or bare `return` | `return E.Variant` |
| Use the value, or a fallback | `x ?: fallback` | `x catch fallback` |
| Pass the empty/failed case upward | `?: return` | `try x` |
| Access a member if present | `x?.member` | — |
| Bind if present | `if (val v = x)` / `while (val v = x)` | `catch \|e\|` |

Seeing `?` tells you "may be missing"; seeing `!` tells you "may fail". The two never share a spelling.

---

## 1a. Why `absent` Is Not `null`

Removing `null` and adding `absent` would be pointless if `absent` failed the same way. It does not, because what made `null` dangerous was never the word. It was four properties, and `absent` has none of them.

| What made `null` dangerous | Java / C | Bestie |
| -------------------------- | -------- | ------ |
| **Every reference may be null.** Nothing in a type says "this one never is" | `User u` may hold `null`; so may every field, parameter, and return | `User` is **never** absent. Only `User ?` may be, and the `?` is in the signature |
| **Using it does not require checking it.** The check is a convention | `u.name` compiles whether or not `u` was checked | `u.name` on a `User ?` is a **compile-time error**. The only ways in are `?:`, `?.`, `if`-let, and `while`-let (§3) |
| **The failure happens far from the cause.** A `null` stored in one place crashes somewhere else, later | A field set to `null` in a constructor throws `NullPointerException` in a different class, minutes later | `absent` cannot be stored in a `T`, passed as a `T`, or returned as a `T`. It stays inside `T ?` until code handles it, at a place the compiler can point to |
| **It is the default for "not set yet".** Uninitialized fields, missing map keys, failed lookups, and errors all become `null` | Java zero-initializes fields to `null`; C leaves garbage | Fields and locals must be assigned before use (`lang.md` §4.7, `oop.md` §11.10). Failures are `T ! E`, not absence. Nothing becomes `absent` unless code says so |

So the `NullPointerException` cannot happen in safe Bestie: there is no operation that takes a `T ?` and uses it as a `T` without handling the absent case. The compiler proves this for every program, the same way it proves types.

### What remains — and why it is different

Three things can still end a program around absence. Each is written in the source, at the point of failure, and none can be reached by accident:

* **`x ?: panic("reason")`.** The programmer states that absence is impossible here, and why. That is an assertion, not a hidden crash: it is searchable, it carries a message, and it fails at the line that made the claim, not three calls later.
* **Raw pointers.** A `ptr<T>` is never the zero address, but nothing checks that its target is alive or holds a `T` (`memory.md` §8). This is the explicit low-level layer. C's `NULL` reaches Bestie only as `ptr<T> ?`, which is absence like any other.
* **`@trusted` and `foreign` code**, where the programmer has taken responsibility in writing.

### Rules that keep it that way

* **No force-unwrap.** There is no `x!!` or `x.get()`. Asserting presence costs a reason (`?: panic("…")`).
* **No implicit conversion** from `T ?` to `T`, ever — not in arguments, returns, assignments, string interpolation, or comparisons.
* **No "null object" default.** A missing value never turns into `0`, `""`, or an empty list by itself. `?: 0` is allowed, but it is written.
* **No absent-by-default.** A `T ?` field still needs an initializer or an `init` assignment; it does not silently start as `absent`.
* **Absence is not failure.** "Not found" is absence. "Could not find out" is an error: `T ! E`, or `T ? ! E` when both are possible (`types.md` §8.4). A function that returns `absent` to mean "something went wrong" hides the reason, which is the other half of what made `null` harmful.

### Guidance against overuse

Kotlin code sometimes makes everything nullable and chains `?.` through it, which brings back the problem in a new shape: a value that is "maybe there" at every layer, and an `absent` at the end that nobody can trace. Use `T ?` only where absence is a normal, expected outcome — a lookup miss, an optional field, the end of an iterator. Unwrap it as close to its source as possible. A long `?.` chain that ends in `?: defaultValue` is usually a missing check that belongs earlier. `bestie lint` warns on `?: panic` in library code and on `T ?` parameters that every caller passes as present.

---

## 2. The Type

```bestie
fun find(id: int): User ?                // a partial function: a user, or nothing
fun connect(host: str, port: int ?)      // an optional parameter
val token: str ?                         // an optional field or binding
```

* `T ?` is core syntax, sealed with the rest of core. There is no `option<T>` or `Optional<T>`; `T ?` is the only spelling.
* `?` binds to the type immediately before it: `list<str ?>` is a list of optional strings, `list<str> ?` an optional list.
* `T ? ?` is legal (an optional optional) but rarely useful; each level costs one niche or one tag (§7).
* `T ? ! E` — a value, a clean absence, or a failure — is legal and costs nothing extra. `T ! E ?` is not a type (`types.md` §8.4).

**Presence needs no constructor.** A `T` is accepted wherever a `T ?` is expected:

```bestie
var token: str ? = absent
token = "abc123"                         // present
```

**`absent`** is the absent value, a contextual keyword valid only where a `T ?` is expected. It is not a null literal: it has no type of its own, cannot be stored in a `T`, cannot be compared with `==`, and cannot be dereferenced. In a function returning `T ?`, a bare `return` means the same as `return absent`.

**Optional is not default.** `port: int = 8080` lets the caller omit the argument and the compiler fill a constant. `port: int ?` means the value may be missing at run time. They are different tools.

---

## 3. Getting the Value Out

A `T ?` can never be used where a `T` is expected. Bestie has no truthiness, no implicit unwrap, and no force-unwrap operator. There are four ways to handle the missing case, and each one is visible where it happens.

### 3.1 `?:` — Value or Fallback

`x ?: y` yields the value of `x` if it is present, and evaluates `y` otherwise. It is Kotlin's *elvis* operator.

```bestie
val port  = cfg.port ?: 8080                       // fallback value
val user  = findUser(id) ?: return                 // absent → return absent from this function
val user  = findUser(id) ?: return ApiError.NotFound  // absent → fail with an error (in a T ! E function)
val line  = it.next() ?: break                     // absent → leave the loop
val entry = try dir.next() ?: break                // T ? ! E: try takes the error, ?: the end
val conf  = loadLocal() ?: loadGlobal() ?: Config.default()   // first present one
val key   = env("API_KEY") ?: panic("API_KEY is not set")
```

Rules:

* The right operand is evaluated **only** when the left is absent.
* If the right operand has type `T`, the result is `T`. If it has type `T ?`, the result is `T ?`, which is what makes chains work.
* The right operand may be an expression that does not complete — `return`, `break`, `continue`, or a call returning `never` such as `panic` (`exceptions.md` §4.4). The result is then `T`.
* `?:` is right-associative and binds more loosely than every operator except assignment (`lang.md` §15.2). `a ?: b + 1` is `a ?: (b + 1)`.
* For a fallback that needs several statements, use `if`-let (§3.3).

### 3.2 `?.` — Access If Present

`x?.member` reads a field or calls a method when `x` is present, and is absent otherwise:

```bestie
val city  = user?.address?.city                    // str ?
val upper = name?.toUpper()                        // str ?
val n     = this.queue?.size()                     // int ?
```

Rules:

* If `x` is absent, the **rest of the postfix chain** is skipped: in `x?.a.b()`, neither `.a` nor `.b()` is evaluated, and the result is absent.
* The result is `U ?` where `U` is the member's type. If the member is already `U ?`, the result is `U ?`, not `U ? ?` — chains of optional fields stay flat.
* Arguments of a skipped call are not evaluated.
* `?.` works on places, with the borrow rules of §5: `this.top?.value` reads the value of an owned node without moving it.
* A member returning `T ! E` cannot be called through `?.`, because `T ! E ?` is not a type (`types.md` §8.4). Bind the optional first, then handle the error: `if (val h = req.header("Content-Length")) { val n = try h.toInt() }`. Absence and failure are kept apart on purpose.

### 3.3 `if`-let — Bind If Present

```bestie
if (val user = findUser(id)) {
    sendEmail(user)                  // user: User
} else {
    log("no user ${id}")
}
```

The binding exists only inside the block. The outer `T ?` is not accessible there.

### 3.4 `while`-let — Repeat While Present

```bestie
while (val job = queue.poll()) {
    run(job)
}
```

The expression is evaluated before every pass, and the loop ends at the first absent value. `while`-let has no `else`. `break` and `continue` work as in any `while` (`lang.md` §13). The `for/in` loop is defined by the same rule: an iterator's `next(): T ?` returning absent ends it.

---

## 4. Why These Spellings

**Why `?:` and not `else`.** Earlier drafts unwrapped with `x else { fallback }`. `?:` replaces it, so there is one spelling per idea and every absence operator contains a `?`. It is also what Kotlin, Groovy, and PHP engineers already read without thinking, and close to `??` in C#, Swift, and JavaScript. `else` belongs to `if` and `when` only.

**Why no force-unwrap (`x!!`).** A force-unwrap is a crash that looks like an operator. In Bestie the crash is written out: `x ?: panic("why this cannot be absent")`. It costs the same, and the reason is in the source.

**Why no `== absent`.** Comparing with `absent` would bring back the "check, then use" pattern that null invites, where the check and the use drift apart. `if`-let checks and binds in one step, so the use cannot happen without the check.

---

## 5. Absence in Places — Fields and Elements

An optional field or element is a *place* that can be empty. Three rules make places safe without a lifetime solver; the details are in `memory.md`.

| Operation | Effect |
| --------- | ------ |
| `if (val x = place)`, `while (val x = place)`, `place?.member` | For a copyable `T`: a copy. For `own T` or a `class`: a **borrow** for the block or the expression. The place cannot be assigned, moved out of, or swapped while borrowed |
| `move place` on an `own T ?` place | Yields its value and leaves the place `absent` (`memory.md` §5.2.1) |
| `place.address()` on a `T ?` place | `ptr<T> ?` — absent stays absent (`memory.md` §8.3) |
| `place = absent` on an `own T ?` place | Allowed only when the place is already empty; otherwise it would leak the object. Use `move` or `swap` |

```bestie
class Node {
    var value: int
    var own next: Node ?                      // absent = end of the list
}

if (val n = this.head) { n.value += 1 }       // borrow; head is unchanged
val own first = move this.head ?: return      // head is now absent
```

`fp.md` §3.5 shows these rules building a linked list, and when to use a count instead of empty array slots.

---

## 6. Absence at the Boundaries

* **FFI.** A C pointer that may be `NULL` is `ptr<T> ?`. Absent is stored as the zero address, so `NULL` crosses in both directions with no conversion (`std-api/foreign.md` §8).
* **Databases.** SQL `NULL` is absence (`std-api/db.md` §5.2).
* **Formats.** A missing JSON field decodes to absent when the target field is `T ?`, and is an error otherwise (`std-lib/format.md`).

---

## 7. Cost

`T ?` costs **nothing** when `T` has a *niche* — a bit pattern no valid `T` uses — and one tag byte otherwise (`memory-layout.md` §11):

| Type | Size of `T ?` | Absent is stored as |
| ---- | ------------- | ------------------- |
| `own T`, a `class` reference, `ptr<T>`, `fn` | One pointer | The zero address |
| `bool` | 1 byte | The value 2 |
| `char` | 4 bytes | A surrogate code point |
| An `enum` | Same as the enum | An unused tag value |
| `int in 0..=200` | 1 byte | The value 201 |
| `int`, `float64` | One extra tag byte (placed in existing padding when there is any) | The tag |

Checking presence is one compare and branch, exactly as a C null check. `?.` chains and `?:` compile to that compare, never to a call.

---

## 8. Summary

* `T ?` is the type, `absent` the empty value, and a `T` is accepted where a `T ?` is expected.
* `?:` gives a fallback or diverts; `?.` accesses if present; `if`-let and `while`-let bind.
* `?` means "may be missing", `!` means "may fail", and the two are never confused.
* There is no null, no force-unwrap, no truthiness, and no `== absent`. A `T ?` can never be used as a `T` without handling the absent case, so a `NullPointerException` cannot happen in safe code (§1a).
* With a niche, absence is free; without one, it costs a tag byte, and the `?` in the type says so.

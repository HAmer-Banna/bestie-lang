# bestie.lib.regex — Regular Expressions

This document defines **`bestie.lib.regex`**, Bestie's regular-expression library: one type, `Regex`, and the small set of types its matches return.

```bestie
import bestie.lib.regex
```

It sits beside `bestie.lib.strings` (`strings.md`) and deliberately does not repeat it. Section 7 draws the line between them.

---

## 1. Design Principles

1. **Linear time, always.** Matching takes time proportional to the pattern size times the input length, for every pattern and every input. There is no backtracking engine, so there is no catastrophic backtracking and no ReDoS. A gateway can match a user-supplied pattern against a user-supplied body without a timeout. This rules out backreferences and lookaround (§3.3).
2. **Compile once, at compile time when possible.** A pattern in a `const` is parsed, checked, and compiled to an automaton by the compiler. A bad pattern is a compile-time error, and the program carries the finished tables in read-only memory (§2.1). This is `vision.md` Rule 2: the convenience comes from the compiler.
3. **Matches are views, not copies.** A match refers to the input it was found in. Getting the matched text allocates nothing (§4). Allocation happens only where a new `str` or `list<str>` is built, and those calls say so in their name.
4. **One spelling per idea.** Flags are written inline in the pattern (`(?i)`). Replacement is a literal `str` or a lambda, never a template language with its own escape rules (§5). Regex operations are methods on `Regex`, never overloads of the `str` operations in `strings.md` (§7).
5. **Freestanding.** The library uses no OS. A `const` regex matches with no allocator at all, so it works in a kernel or on a microcontroller.

---

## 2. Creating a `Regex`

`Regex` is a `class`. A compiled pattern has a size that depends on the pattern, so it is not a value type.

### 2.1 Compile-Time Patterns — `const`

```bestie
const DATE  = Regex.compile("""(\d{4})-(\d{2})-(\d{2})""")
const IDENT = Regex.compile("""[A-Za-z_][A-Za-z0-9_]*""")
```

`Regex.compile` is `@pure`, so a call whose argument is a constant is evaluated by the compiler (`lang.md` §4.1). The result is a constant object in `.rodata`, as a `const` collection is (`collections.md` §5.2):

* A syntax error, or a feature outside §3, is a **compile-time error** at the declaration.
* The pattern is compiled to a **full DFA** and stored in `.rodata`. Matching is a table walk with no allocation and no warm-up.
* If the full DFA would exceed the compiler's size limit (the pattern is too large to precompute), the declaration is a compile-time error that names the limit. Make that pattern a runtime `Regex` instead (§2.2).
* A `const` regex is immutable and may be shared across threads with no lock.

Write patterns as raw strings (`"""…"""`, `lang.md` §3.1.6). Raw strings process no escapes and no interpolation, so `\d` reaches the regex engine as written, with no doubled backslashes.

### 2.2 Runtime Patterns

```bestie
fun Regex.compile(pattern: str): own Regex ! RegexError
fun Regex.compile(pattern: str, limits: RegexLimits): own Regex ! RegexError
```

A pattern known only at run time, from configuration or a user, is compiled when the program reaches it:

```bestie
val own filter = Regex.compile(userPattern) catch |e| return Response.badRequest(e.toStr())
defer filter.free()

if (filter.isMatch(line)) { emit(line) }
```

A runtime `Regex` matches with a **lazy DFA**. It builds DFA states as the input needs them and keeps them in a cache owned by the `Regex`. The cache size is bounded by `RegexLimits`, and memory comes from the current allocator (`allocators.md`). In freestanding code with no allocator registered, `Regex.compile` with a non-constant argument is a compile-time error (`core/freestanding.md` B2), and only `const` regexes are available.

```bestie
value class RegexLimits {
    val patternSize: int = 64 * 1024     // compiled program, in bytes
    val cacheSize:   int = 2 * 1024 * 1024
}
```

A pattern whose compiled program exceeds `patternSize` fails with `RegexError.TooLarge`. Set the limits low for patterns that come from untrusted input. When the cache fills, the engine clears it and continues. Matching slows down but stays correct and linear.

### 2.3 Errors

```bestie
errors RegexError {
    Syntax,          // malformed pattern
    Unsupported,     // a feature excluded by §3.3, such as a backreference
    TooLarge,        // compiled program over RegexLimits.patternSize
}
```

`e.toStr()` includes the byte offset in the pattern and what was expected. Matching itself never fails.

---

## 3. Pattern Syntax

The syntax is the common core of RE2, Rust `regex`, and Go `regexp`, so patterns move between those ecosystems unchanged.

### 3.1 Supported

| Form | Meaning |
| ---- | ------- |
| `.` | Any scalar except `\n` (with `(?s)`: any scalar) |
| `[abc]` `[^abc]` `[a-z]` | Character classes |
| `\d` `\w` `\s` | ASCII digit, ASCII word character (`[A-Za-z0-9_]`), ASCII whitespace |
| `\D` `\W` `\S` | Their negations |
| `\p{L}` `\p{Greek}` `\P{…}` | Unicode general category or script, and its negation |
| `^` `$` | Start and end of input (with `(?m)`: of each line) |
| `\b` `\B` | ASCII word boundary, and its negation |
| `*` `+` `?` `{n}` `{n,}` `{n,m}` | Greedy repetition |
| `*?` `+?` `??` `{n,m}?` | Lazy repetition |
| `a\|b` | Alternation |
| `(…)` | Capture group |
| `(?<name>…)` | Named capture group |
| `(?:…)` | Non-capturing group |
| `(?i)` `(?m)` `(?s)` `(?x)` | Flags: case-insensitive, multi-line, dot-matches-newline, verbose (whitespace and `#` comments ignored) |
| `(?i:…)` | Flags scoped to a group |
| `\` + punctuation, `\n` `\t` `\u{…}` | Escapes |

`\d`, `\w`, `\s`, and `\b` are **ASCII**, matching `char.isDigit()` in `strings.md` §8a. A backend parsing a port number or an HTTP header almost never wants `\d` to accept Arabic-Indic digits. For Unicode, ask by name: `\p{Nd}` for any decimal digit, `\p{L}` for any letter.

### 3.2 Matching Semantics

* Input is UTF-8 `str`. The engine matches **Unicode scalars**: `.` never matches half of a multi-byte character.
* Leftmost-first semantics: among matches starting at the same position, the one the pattern prefers wins (as in Perl, RE2, and Rust). Greedy and lazy repetition behave as programmers expect.
* `(?i)` uses Unicode simple case folding, with no locale (`strings.md` §7).

### 3.3 Excluded — and Why

| Feature | Why it is excluded |
| ------- | ------------------ |
| Backreferences (`\1`) | No linear-time algorithm exists. They are the main source of catastrophic backtracking |
| Lookahead / lookbehind (`(?=…)`, `(?<=…)`) | Same reason. Use a capture group and check the context in code |
| Possessive quantifiers, atomic groups | Only meaningful in a backtracking engine |
| Recursion, conditionals | Not regular; a parser is the right tool |
| A literal syntax (`/…/`, `re"…"`) | A `const` with a raw string already gives compile-time checking, so core needs no new syntax |

An excluded feature in a `const` pattern is a compile-time error. In a runtime pattern it is `RegexError.Unsupported`. It is never silently treated as a literal.

---

## 4. Matching

### 4.1 Operations

| Method | Returns | Allocates |
| ------ | ------- | --------- |
| `re.isMatch(s: str)` | `bool` | ❌ |
| `re.find(s: str)` | `Match ?` — the leftmost match | ❌ |
| `re.findAll(s: str)` | lazy iterator over `Match`, non-overlapping | ❌ |
| `re.captures(s: str)` | `Captures ?` — the leftmost match with its groups | ❌ |
| `re.capturesAll(s: str)` | lazy iterator over `Captures` | ❌ |
| `re.matchesWhole(s: str)` | `bool` — the pattern matches the entire input, as if anchored | ❌ |
| `re.split(s: str)` | lazy iterator over `str` pieces between matches | ❌ |
| `re.replaceFirst(s, …)` / `re.replaceAll(s, …)` | `str` (§5) | ✅ the result |

`isMatch` is the fastest form: it stops at the first match, and it skips capture tracking entirely. Use it when you need only yes or no.

The iterators are lazy and allocate nothing; they are walked with `for/in`. To keep the results, collect them by name: `re.split(s).toList()` (`collections.md` §9). The cost is visible at the call site (`vision.md` Rule 3).

```bestie
for (m in IDENT.findAll(source)) {
    symbols.add(m.text)
}
```

### 4.2 `Match` and `Captures`

`Match` and `Captures` are small value types. Their text is a `str` that shares the input's buffer, which is always safe because `str` is immutable (`strings.md` §3), so a match may be kept after the input binding is gone.

```bestie
value class Match {
    val text:      str     // the matched text — shares the input's buffer, no copy
    val byteStart: int     // byte offset of the first matched byte
    val byteEnd:   int     // byte offset one past the last matched byte
}
```

Offsets are **byte** offsets, and the names say so. Every regex engine works in bytes, and a byte offset converts to a core slice in O(1): `s[m.byteStart..m.byteEnd]`. The codepoint indices used by `strings.md` would cost O(n) for every match. When you need text, use `m.text`, which is already a `str`.

| `Captures` member | Returns | Notes |
| ----------------- | ------- | ----- |
| `c[0]` | `str` | The whole match — always present |
| `c[i]` | `str ?` | Group `i`; absent if the group did not take part in the match. Panics if `i` exceeds the number of groups |
| `c["name"]` | `str ?` | Named group. Panics if the pattern has no group with that name |
| `c.group(i)` / `c.group(name)` | `Match ?` | The group with its byte offsets |
| `c.count` | `int` | Number of groups, including group 0 |

A group that did not participate is `absent`, never `""`. An empty match and no match are different results.

```bestie
if (val c = DATE.captures(line)) {
    val year  = c[1] ?: return      // str ? → str
    val month = c[2] ?: return
    log("${year}/${month}")
}
```

Named groups read the same way:

```bestie
const KV = Regex.compile("""(?<key>\w+)=(?<value>\S*)""")

if (val c = KV.captures(line)) {
    val key   = c["key"]   ?: return
    val value = c["value"] ?: ""
}
```

---

## 5. Replacement

```bestie
fun Regex.replaceFirst(s: str, with: str): str
fun Regex.replaceAll(s: str, with: str): str
fun <F impl fn(Captures) -> str> Regex.replaceFirst(s: str, with: F): str
fun <F impl fn(Captures) -> str> Regex.replaceAll(s: str, with: F): str
```

* A `str` replacement is **literal**. `$1` in it is a dollar sign and a one, nothing more.
* To use the groups, pass a lambda and build the text with core string interpolation:

```bestie
// 2026-11-01 → 01/11/2026
val us = DATE.replaceAll(text, c => "${c[3]}/${c[2]}/${c[1]}")

// mask all but the last four digits of every card-like number
const CARD = Regex.compile("""\d{12}(\d{4})""")
val masked = CARD.replaceAll(log, c => "************${c[1]}")
```

Other libraries have a replacement template language (`$1`, `${name}`, `\1`) with its own escaping rules, and it clashes with Bestie's `${…}` interpolation. Interpolation already exists in core, a lambda is inlined because it is a generic `F` (`fp.md`), and so there is nothing for a template language to add. If the regex never matches, the result is the input `str` itself, with no copy.

---

## 6. Matching Bytes

Protocol parsers, log scanners, and binary formats often hold bytes that are not valid UTF-8. `Regex.compileBytes` builds a regex that matches a `slice<byte>`:

```bestie
const STATUS = Regex.compileBytes("""^HTTP/1\.[01] (\d{3}) """)

if (val c = STATUS.captures(buf[..n])) { … }
```

In a bytes regex, `.` and negated classes match **any byte**, Unicode classes (`\p{…}`) are an error, and matches return `slice<byte>` instead of `str`. Everything else — linear time, `const` compilation, views not copies — is the same. A bytes regex and a text regex are different types, so passing a `str` to one built for bytes is a compile-time error.

---

## 7. Overlap with `bestie.lib.strings`

Both libraries search, split, and replace. The rule is simple: **a `str` argument is always literal, and a `Regex` is always a pattern.** Nothing in Bestie interprets a `str` as a regex.

That rules out Java's best-known trap — `"a.b".split(".")` returns an empty array, because Java's `split` reads its argument as a regex. In Bestie, `"a.b".split(".")` is `["a", "b"]`, and splitting on a pattern means writing a `Regex`.

| Task | Use | Why |
| ---- | --- | --- |
| Does `s` contain `"error"`? | `s.contains("error")` | A literal search is a vectorized `memchr`-style scan. It is faster than any regex and allocates no automaton |
| Split on `","` | `s.split(",")` | Literal separator |
| Split on runs of whitespace | `s.splitWhitespace()` | Common enough to have its own name |
| Split on `,` or `;` with optional spaces | `SEP.split(s)` with `const SEP = Regex.compile("""\s*[,;]\s*""")` | A pattern |
| Replace `"\r\n"` with `"\n"` | `s.replace("\r\n", "\n")` | Literal |
| Rewrite dates from one format to another | `DATE.replaceAll(s, c => …)` | Structure plus groups |
| Is `s` a number? | `s.toInt()` (`strings.md` §2) | Parses and range-checks in one step; `\d+` does neither |
| Is `c` a digit? | `c.isDigit()` (`strings.md` §8a) | One comparison |
| Validate an identifier or a token shape | `IDENT.matchesWhole(s)` | A pattern |
| Extract fields from a log line | `LINE.captures(s)` | Groups |

Rules for the two libraries:

* `bestie.lib.strings` never gains a regex overload. Its functions take `str` needles and stay literal.
* `bestie.lib.regex` never duplicates a literal operation. Where a literal is enough, the table above points to `strings.md`.
* Internally, the regex engine uses literal search for patterns that start with, or consist of, a literal. `Regex.compile("""error""")` is as fast as `contains("error")`. The choice between the two is about meaning, not speed.
* `strings.md` uses **codepoint** indices; this library returns **byte** offsets, named `byteStart`/`byteEnd` (§4.2). Use `m.text` to get from one to the other without counting.

---

## 8. Performance

| Operation | Cost |
| --------- | ---- |
| `const` compilation | Compile time only. Zero at run time |
| `Regex.compile` at run time | O(pattern size), once. Allocates the program and a bounded cache |
| `isMatch`, `find` | O(input length), no allocation |
| `captures` | O(input length × groups) in the worst case. The DFA finds the match, then a bounded second pass over only the matched text recovers the groups |
| `findAll`, `split` | O(input length) total, lazy, no allocation |
| `replaceAll` | O(input length) plus one allocation for the result |

---

## 9. Intentional Non-Features

* No backtracking engine, backreferences, or lookaround (§3.3)
* No regex interpretation of `str` arguments anywhere in the standard library (§7)
* No replacement template language (§5)
* No global or implicit regex cache: a pattern is compiled where it is written, once
* No literal syntax in core: `const` plus a raw string is the literal
* No locale-dependent matching

---

## 10. Summary

* `Regex` is linear-time for every pattern and every input, so it is safe on untrusted data.
* A `const` regex is checked and compiled by the compiler, stored in `.rodata`, and matched with no allocation — usable in freestanding code.
* Matches are views with byte offsets; text comes out as `str` with no copy.
* Replacement uses a literal `str` or a lambda with core interpolation.
* `strings.md` handles literals, `regex.md` handles patterns, and no `str` is ever silently treated as a pattern.

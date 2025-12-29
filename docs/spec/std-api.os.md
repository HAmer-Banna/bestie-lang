1. Purpose and Scope

The std-api.os module defines Bestie’s interaction surface with the operating system.

This module models OS concepts, not data flow and not application logic.
It provides typed, explicit, override-friendly abstractions for:

Filesystem metadata

Processes and execution

Environment variables

Signals and OS events

Time sources backed by the OS

Users, permissions, and exit control

This module does not provide:

File content IO (handled by std-api.io)

Networking

HTTP

Databases

Containers or virtualization

2. Design Philosophy
2.1 Object-first OS Modeling

Operating-system entities have:

Identity

Lifetime

State

Side effects

Therefore, OS concepts are modeled as classes or protocols by default.

Free functions are permitted only when:

The operation is stateless

There is no identity

Overriding or substitution would be meaningless

This rule is mandatory and intentional.

2.2 Avoiding the Python and Java Traps

Bestie explicitly avoids:

Mixing half the API as functions and half as classes

Multiple competing abstractions for the same concept

Layered redesigns (IO, NIO, NIO2, etc.)

Every OS concept has:

One primary abstraction

One obvious extension point

2.3 OOP and FP Are Tools, Not Ideologies

OOP is used for structure, identity, and extensibility

FP is used for expression, callbacks, and local logic

Lambdas, concise bodies, and method references are encouraged inside methods, not as the public OS API surface.

3. Filesystem (Metadata Layer Only)

File content reading/writing belongs to std-api.io.

3.1 Path (class)

Represents a filesystem path.

Responsibilities:

Normalization

Resolution

Comparison

Platform abstraction

class Path


Rules:

No IO side effects

No implicit filesystem access

Immutable

3.2 FileInfo (value class)

Immutable snapshot of filesystem metadata.

Contains:

Size

Permissions

Timestamps

Ownership

File kind (file, directory, link, etc.)

value class FileInfo

3.3 FileSystem (protocol)

Primary filesystem abstraction.

Responsibilities:

Metadata queries

Creation and deletion

Directory traversal

protocol FileSystem


Benefits:

Enables in-memory filesystems

Enables test doubles

Enables OS-specific implementations

4. Processes and Execution
4.1 Process (class)

Represents a running or completed OS process.

Responsibilities:

PID

Exit status

Lifecycle state

class Process


Rules:

No global process state

No static process control functions

4.2 ProcessBuilder (class)

Explicit construction of a process.

Responsibilities:

Command specification

Environment injection

IO wiring (via std-api.io)

Working directory

class ProcessBuilder


Rationale:

No hidden defaults

No shell magic

Fully testable

4.3 Process Control

Operations:

start

wait

terminate

signal

Signals are typed, not integers.

5. Environment Variables
5.1 Environment (class)

Represents an environment scope.

Capabilities:

Read-only view

Mutable scoped view

Snapshot vs live behavior

class Environment


Rules:

No global getenv

Explicit access required

Safe for testing

6. Signals and OS Events
6.1 Signal (enum)

Strongly typed representation of OS signals.

enum Signal


Mapped per platform at compile time.

6.2 Signal Handling

Handlers registered via methods

Lambdas permitted

Lambdas must not capture state

Signature verified at compile time

No implicit signal hooks.

7. Time and Clocks (OS-backed)
7.1 Clock (protocol)

Represents a source of time.

Kinds:

Monotonic

Wall-clock

protocol Clock

7.2 Instant and Duration (value classes)

Used consistently across:

OS APIs

std-lib datetime

Scheduling

No primitive timestamps.

8. Users, Groups, and Permissions
8.1 User and Group (value classes)

Represent OS users and groups.

Contains:

IDs

Names

Resolution APIs

Immutable.

8.2 Permissions (value class)

Explicit permission modeling.

No raw integers

No bitwise user code

Platform-mapped internally

9. Exit and System Control
9.1 ExitCode (value class)

Strongly typed exit status.

value class ExitCode

9.2 Controlled Termination

Explicit termination APIs

Testable

No hidden shutdown hooks

10. Error Model

OS errors are:

Typed

Explicit

Non-string-based

No unchecked global failures.

11. What This Module Intentionally Does NOT Provide

This module does not include:

Shell scripting DSLs

Process pipelines as syntax

Platform-specific hacks

Auto-parallel execution

Async abstractions (handled by concurrency APIs)

12. Extension Policy

New OS-related features:

Go into std-api.os only if general-purpose

Otherwise live in external libraries

This keeps the API:

Small

Predictable

Long-lived

13. Rationale: Practical Language Design

Bestie:

Is not anti-OOP

Is not FP-only

Is not minimal for the sake of minimalism

Bestie is practical:

Structured

Explicit

Fast

Safe

Extensible without chaos

This module reflects that philosophy.

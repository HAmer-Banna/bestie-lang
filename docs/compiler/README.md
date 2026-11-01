# Bestie Compiler

This directory contains the reference compiler for the Bestie Programming Language.

Stages:

- Lexer
- Parser
- AST
- Type checker
- Memory model checker
- Concurrency checker
- IR generation
- Code generation (native)

Documents:

- `compiler-architecture.md` — pipeline, IRs, optimization strategy, layout obligations
- `codegen-advantages.md` — why Bestie can produce faster code and smaller data than C, and the rule behind each advantage

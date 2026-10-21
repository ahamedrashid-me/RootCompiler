# RootCompiler

> **The mother of compilers.** A small, self-hosting, dependency-minimal compiler toolkit in which the *language*, its *syntax*, and its *memory model* are all replaceable parts.

| | |
|---|---|
| **Project** | RootCompiler |
| **CLI** | `rcom` |
| **Reference language** | RootCode (`.rco` source files, *proposed extension*) |
| **Intermediate representation** | RIR (Root Intermediate Representation) |
| **Reference memory model** | RMM-0 (Root Memory Model, baseline) |
| **Backend** | LLVM (textual IR → `llc` → `lld`) |
| **Document status** | **Design draft v0.1. Nothing described here is implemented yet.** |

---

## Table of contents

1. [Status, conventions, and how to read this document](#1-status-conventions-and-how-to-read-this-document)
2. [What is RootCompiler?](#2-what-is-rootcompiler)
3. [Why is it needed?](#3-why-is-it-needed)
4. [What it can achieve (goals) and what it will not do (non-goals)](#4-goals-and-non-goals)
5. [Design principles](#5-design-principles)
6. [The three phases and the release plan](#6-the-three-phases-and-the-release-plan)
7. [Architecture](#7-architecture)
8. [The RootCode language](#8-the-rootcode-language)
9. [Formal grammar](#9-formal-grammar)
10. [The memory model (RMM-0)](#10-the-memory-model-rmm-0)
11. [Type system](#11-type-system)
12. [RIR: the Root Intermediate Representation](#12-rir-the-root-intermediate-representation)
13. [LLVM backend and platform layer](#13-llvm-backend-and-platform-layer)
14. [Standard library layers](#14-standard-library-layers)
15. [Phase 3: the language-maker toolkit](#15-phase-3-the-language-maker-toolkit)
16. [Diagnostics](#16-diagnostics)
17. [Testing and verification strategy](#17-testing-and-verification-strategy)
18. [Dependency policy](#18-dependency-policy)
19. [Limitations and risks](#19-limitations-and-risks)
20. [Decision register and open questions](#20-decision-register-and-open-questions)
21. [Example programs](#21-example-programs)
22. [Appendices](#22-appendices)

---

## 1. Status, conventions, and how to read this document

This is a **design specification written before the implementation**. To keep it honest, every section is implicitly one of three kinds:

| Label | Meaning |
|---|---|
| **Decided** | Chosen by the project owner. Changing it requires an explicit decision-register entry (§20). |
| **Proposed** | A concrete design that is the current best guess. Expected to change when implementation teaches us something. |
| **Open** | Not yet decided. Listed in §20. |

Where a section is a **sketch** or **illustrative**, it communicates direction only and is not a stable interface.

Conventions:

- *Must*, *must not*, *should* are used in their usual specification sense.
- Grammar is written in an ISO-style EBNF: `{ x }` repetition, `[ x ]` optional, `|` alternation, `( )` recording, `"x"` terminal, `(* ... *)` comment.
- "Phase" refers to the three development phases (§6). "v0.x / v1.0 / v2.0" refer to releases.

---

## 2. What is RootCompiler?

RootCompiler is a **compiler toolkit** whose end goal is that *anyone can build their own programming language* — choosing their own syntax, style, features, and **memory model** — and get native code on any LLVM-supported target, without writing a backend and without depending on a heavyweight host language.

It is built in three phases that deliberately reinforce each other:

1. **Phase 1:** a working compiler for the reference language **RootCode**, written in **D without the garbage collector**, generating **LLVM IR from day one**.
2. **Phase 2:** the compiler is **rewritten in RootCode and compiles itself** (self-hosting). After this, the D implementation is no longer needed.
3. **Phase 3:** the internals (syntax definition, semantic rules, memory model, lowering) are exposed as a **public extension mechanism**, so new languages and domain-specific languages (DSLs) can be built *with* RootCompiler. RootCode itself is re-expressed as just another language built with that mechanism.

RootCompiler is therefore three things at once:

- a **compiler** (RootCode → native code),
- a **systems language** (RootCode, designed first to write compilers, then systems software),
- a **language workbench** (the toolkit for making other languages).

---

## 3. Why is it needed?

### 3.1 The problem

Building a new language today forces an unpleasant choice:

| Path | Cost |
|---|---|
| Write your own backend (instruction selection, register allocation, object files) | Years of work per architecture |
| Use LLVM through its C++ API | You need C++ (or bindings in another language), a large build, and unstable APIs |
| Use a language workbench (e.g. Racket, JetBrains MPS, Spoofax) | You inherit a large runtime (a VM, the JVM) and its deployment story |
| Use an IR framework (e.g. MLIR) | Powerful, but a large C++ codebase that expects you to already be a compiler engineer |

On top of that, **the memory model is welded into every language.** Rust's ownership, Swift's ARC, Go's GC, and C's manual management are each inseparable from the compiler that implements them. A researcher or domain expert who wants to try a *different* memory discipline must fork an entire compiler.

### 3.2 What RootCompiler proposes

- **One small, self-hosted compiler** whose source is written in the language it compiles, so the whole toolchain can be understood, built, and ported by one person.
- **LLVM as the only dependency**, so every LLVM target is reachable without writing backends, and nothing else (no C compiler, libc, other languages, or runtimes) is needed after bootstrap.
- **A neutral core IR (RIR)** that contains no memory policy; memory models are **plugins** that analyze and rewrite RIR.
- **A reference language built with the same mechanism users will use**, so the mechanism is proven by use, not by claim.

### 3.3 Relationship to existing work (honest positioning)

RootCompiler does **not** claim to replace these; it overlaps with parts of each.

| Existing work | What it is | Overlap / difference |
|---|---|---|
| **LLVM** | Optimizing compiler backend and toolchain | RootCompiler *uses* it as the sole backend. |
| **MLIR** | Multi-level IR framework with user-defined "dialects" (part of LLVM, C++) | Closest conceptual relative of RIR plus plugins. RootCompiler aims to be far smaller, self-hosted, and without a C++ dependency. |
| **Racket (`#lang`)** | Language-oriented programming on the Racket runtime | Same ambition (many languages, one framework); RootCompiler targets native code and no VM. |
| **JetBrains MPS / Spoofax / Rascal** | Language workbenches (typically JVM-based) | Similar workbench goals; RootCompiler targets self-hosted native tooling. |
| **Nanopass framework** | Compiler built from many small passes | RootCompiler adopts the same pass-oriented philosophy. |
| **QBE / Cranelift** | Small code generators | Alternative backends; RootCompiler chose LLVM for target breadth. |
| **Zig, Rust, Hylo, Vale** | Systems languages with distinct memory strategies | Sources of prior art for RMM; RootCompiler's novelty goal is *making the strategy pluggable*. |

---

## 4. Goals and non-goals

### 4.1 Goals

- **G1. Self-hosting.** The RootCode compiler compiles itself; the fixed-point test (§17) passes.
- **G2. Near-zero dependencies.** After bootstrap, only LLVM (§18).
- **G3. Portability.** Output for any LLVM-supported target; cross-compilation without a target sysroot (no libc).
- **G4. Memory safety by default** in safe RootCode, with deterministic destruction (no GC) and few, explainable rules.
- **G5. Pluggable memory models** (Phase 3): the reference model is a plugin, not baked into the front end.
- **G6. Language-maker toolkit** (Phase 3): grammar, AST, semantic rules, and lowering definable by users, with diagnostics and tooling generated where possible.
- **G7. Understandable.** The compiler is small enough to read; the spec is the source of truth and is tested against the implementation.

### 4.2 Non-goals (for the stable 2.0 release)

- Replacing LLVM, or writing a competitive optimizer. (Revisit only after widespread adoption; see §18.)
- Being the fastest-to-compile or most feature-rich language.
- Closures, threads, async, and a full trait system in 1.0 (see §19).
- A JIT or interpreter.
- IDE integration beyond basic diagnostics and a formatter, before 2.0.
- Binary compatibility with C++ or Rust ABIs.

---

## 5. Design principles

1. **Self-hosting is the acceptance test.** If a feature isn't needed to compile the compiler, it waits.
2. **One bracket, one job.** `()` groups and calls, `[]` indexes and types arrays, `{}` delimits blocks and literals. Ambiguity is a bug in the grammar.
3. **Mechanism over policy.** Memory management, syntax, and type rules are policies; the core provides mechanisms (IR, passes, hooks).
4. **Prove it on yourself.** Every extension point must be used to build RootCode itself.
5. **Safety checks are removed only when proven redundant**, never silently disabled by build mode.
6. **No hidden runtime.** Everything a program needs is visible as library or platform code written in RootCode.
7. **Small and explicit beats large and clever.** Few rules, each with a stable error code and message.
8. **Honest limits.** Every known hole is documented (§19), not hidden.
9. **Cut features, not the goal.** If the schedule slips, scope shrinks; self-hosting does not.

---

## 6. The three phases and the release plan

### 6.1 Phases (Decided)

| Phase | Deliverable | Implementation language | Ends when |
|---|---|---|---|
| **1** | RootCode compiler emitting LLVM IR | **D, without GC** | RootCode programs compile to native binaries; RMM-0 checker enforced by test suite |
| **2** | Self-hosted compiler | **RootCode** | Fixed-point (stage2 == stage3) and the D implementation is retired |
| **3** | Language-maker toolkit | RootCode | A second, *different* language is built with only the public mechanism |

### 6.2 Release plan (Proposed)

| Release | Contents | Exit criterion |
|---|---|---|
| **0.1** | Lexer, parser, `i32`/`bool`, functions, `if`/`while`, records; textual LLVM IR; hello world | A program prints via `rt` write syscall on Linux x86_64 |
| **0.2** | Enums, `match`, `Option`/`Result`, `?` | A small expression evaluator runs |
| **0.3** | RMM-0 checker, drop elaboration, `defer`, error codes E1–E9 | All memory-model conformance tests pass |
| **0.4** | `unsafe`, intrinsics, `Box`, generics, slices/`str` | `Vec<T>` and `String` implemented in RootCode |
| **0.5** | Modules, standard library layers, platform layer, multi-target | Cross-compile to aarch64 and wasm32 |
| **0.9** | Everything the compiler needs | The lexer and parser are rewritten in RootCode and pass the same tests |
| **1.0** | **Self-hosted** | Fixed point; bootstrap seed (`.ll`) checked in; D retired |
| **1.x** | Optimizations, debug info, formatter, more targets | Stability and performance |
| **2.0** | **Language-maker toolkit stable** | A second language is built in days using only public extension points |

### 6.3 Kill criterion

> If the RootCode lexer cannot be compiled by RootCode within **12 months** of starting Phase 1, cut features, not the goal.

Cut order: `match`/sum types sugar → generics (monomorphic duplicates in stdlib) → modules (single file) → everything not on the compiler's critical path. **Never cut the memory-safety core**, since it is what the project exists to explore.

---

## 7. Architecture

### 7.1 Pipeline

```
 source (.root)
     │  lexer
     ▼
   tokens
     │  parser            (grammar §9; error recovery at ';' and '}')
     ▼
    AST   ── every node carries a source span
     │  name resolution + module graph
     ▼
 resolved AST
     │  type checking + monomorphization of generics
     ▼
  typed HIR
     │  lowering (control flow → CFG, drops/defers made explicit as abstract ops)
     ▼
  RIR-0   (abstract ownership operations: move, copy, drop_if_live, borrow…)
     │  memory-model plugin: analyze (check rules, report errors) + elaborate
     ▼
  RIR-1   (concrete: explicit drop calls, drop flags, no ownership ops left)
     │  RIR → LLVM textual IR (allocas; LLVM's mem2reg builds SSA)
     ▼
  .ll files
     │  llc  →  object files
     │  lld  →  executable / library
     ▼
  native binary
```

`llc` and `lld` are part of the LLVM project; no other tool is invoked.

### 7.2 Layering (the key structural decision)

```
Layer 3  Backend         RIR → LLVM IR text; invokes llc, lld
Layer 2  Semantic mods   Type system rules, MEMORY MODEL, effects…
                         (plugins that analyze and rewrite RIR)
Layer 1  Core IR (RIR)   Small, typed, CFG-based, no memory policy
Layer 0  Frontends       Syntax → AST → HIR → RIR (RootCode is one frontend)
```

**Rule:** nothing in Layers 0 or 1 may hard-code a memory discipline. The parser and type checker know *that* a value has an owner or a borrow convention only through hooks the memory model provides. This is what makes Phase 3 possible without a rewrite.

### 7.3 Compiler passes

| # | Pass | Input → Output | Notes |
|---|---|---|---|
| 1 | Lex | text → tokens | Spans, no context-sensitive tokens |
| 2 | Parse | tokens → AST | LL(1)-friendly grammar; multi-error recovery |
| 3 | Resolve | AST → resolved AST | Modules, imports, visibility |
| 4 | Typecheck | → typed HIR | Local inference; explicit signatures; generics instantiated here |
| 5 | Lower | HIR → RIR-0 | structured control flow → CFG |
| 6 | Memory model: analyze | RIR-0 | Dataflow; emits E-codes |
| 7 | Memory model: elaborate | RIR-0 → RIR-1 | Inserts drops and drop flags |
| 8 | RIR verifier | RIR-1 | Internal consistency checks |
| 9 | Emit | RIR-1 → `.ll` | One LLVM module per RootCode module |
| 10 | Link | `.ll` → binary | `llc`, `lld` |

Each pass is a function over an explicit data structure, so passes can be tested in isolation (nanopass style).

### 7.4 Memory discipline *of the compiler itself*

The compiler is the first and hardest program RootCode must run:

- **Arena per phase** (parse arena, type arena, RIR arena), freed wholesale.
- AST and IR are **owned trees**; cross-references (symbol → declaration, use → definition) use **typed index handles** (`NodeId`, `SymbolId`) into arenas/vectors, not pointers.
- The Phase 1 D implementation follows the *same* discipline (`@nogc`, own arena and containers, no Phobos), so Phase 2 is largely transliteration.

### 7.5 Repository layout (Proposed)

```
rootcompiler/
  docs/
    ROOTCOMPILER.md       this document
    DECISIONS.md          §20 register, kept in sync
    GRAMMAR.md            §9, generated from the grammar test file
  bootstrap/
    seed/                 checked-in .ll of the self-hosted compiler (from 1.0)
  compiler-d/             Phase 1 implementation (D, @nogc)
    lexer.d  parser.d  ast.d  resolve.d  typecheck.d
    lower.d  rir.d  memmodel_rmm0.d  emit_llvm.d  driver.d
  compiler/               Phase 2+ implementation (RootCode)
  lib/
    core/  alloc/  os/  rt/
  tests/
    grammar/  typecheck/  memmodel/  codegen/  conformance/  fuzz/
  tools/
    rcom-fmt/
```

### 7.6 CLI (Proposed)

| Command | Purpose |
|---|---|
| `rcom build <file>` | Compile and link |
| `rcom run <file>` | Build and execute |
| `rcom check <file>` | Parse, typecheck, and run memory checks only |
| `rcom emit-llvm <file>` | Print LLVM IR |
| `rcom emit-rir <file>` | Print RIR-0 / RIR-1 (debugging) |
| `rcom test` | Run tests in the project |
| `rcom --target=<triple>` | Cross-compile |
| `rcom --explain E1` | Long explanation for an error code |

---

## 8. The RootCode language

RootCode is a statically typed, ahead-of-time compiled systems language designed first for writing compilers.

### 8.1 Feature summary

| Area | v1.0 | Later |
|---|---|---|
| Types | Fixed-width ints, floats, `bool`, `char`, tuples, fixed arrays, records, sum types (`enum`), `Box<T>`, `Vec<T>`, `String`, projection types `str` and `[T]` | Traits/interfaces, associated types |
| Generics | Monomorphized, unconstrained (checked at instantiation) | Trait bounds |
| Memory | RMM-0: owned values, moves, deterministic drop, parameter conventions | Generational handles, regions, pluggable models (Phase 3) |
| Control | `if`, `while`, `loop`, `for` (ranges/iterators), `match`, `stop`, `run`, `get`, `defer` | Labeled stops |
| Errors | `Result<T, E>`, `Option<T>`, `?` | Error traits, stack traces |
| Modules | One file = one module, `import`, `pub` | Packages, versioning |
| Unsafe | `unsafe` blocks, raw pointers, intrinsics (for the runtime and library only, by policy) | Audited unsafe lint |
| Not in 1.0 | Closures, threads, async, macros, comptime, FFI beyond `extern` declarations | See §19 |

### 8.2 Primitive types

| Type | Meaning | LLVM |
|---|---|---|
| `i8 i16 i32 i64` | Signed integers | `i8 i16 i32 i64` |
| `u8 u16 u32 u64` | Unsigned integers | same width |
| `isize usize` | Pointer-sized integers | target pointer width |
| `f32 f64` | IEEE floats | `float`, `double` |
| `bool` | Boolean | `i1` (stored as `i8`) |
| `char, str` | Unicode scalar value eg char letter = 'a'; and str text = "hello"; | `i32` |
| `()` | Unit | empty record / none |
| `never` | Type of expressions that don't get | `unreachable` |

> Migration note: RasCode's `int` → choose a width (`i32`/`i64`); `deci` → `f64`; `byte`/`ubyte` → `i8`/`u8`; `none` → `()`.

### 8.3 Semantics highlights

- **Integer arithmetic is checked by default** (trap on overflow) in all build modes, unless proven redundant by the compiler or explicitly requested (`wrapping_add`, etc.).
- **No null.** Absence is `Option<T>`.
- **Evaluation order** is left to right.
- **Array and `Vec` indexing is bounds-checked** unless proven in range.
- **Casts** use `as` and are explicit between numeric types; narrowing casts truncate, and checked conversions are library functions.
- **Panics** call the runtime panic handler (`rt_panic`), which by default writes a message and exits nonzero. No unwinding in 1.0.
- **Methods** are free functions with uniform call syntax: `a.f(b)` means `f(a, b)` (D-style UFCS). A function-pointer *field* is called as `(a.f)(b)`.
- **Generics instantiation:** `max::<i32>(a, b)` in expressions, `Vec<i32>` in types. The turbofish form is required only when inference cannot decide.

### 8.4 Modules

One file is one module. The module path follows the directory structure.

```
import os::io;                 // use as io::println(...)
import util::math as m;        // alias
pub fnc area(r: f64) :: f64 { ... }
```

Name resolution order: local scope → module items → explicit imports → the prelude (`Option`, `Some`, `None`, `Result`, `Ok`, `Err`, `Box`, `Vec`, `String`).

### 8.5 Parameter conventions

Every parameter has one of four conventions (adapted from Mutable Value Semantics as in Hylo):

| Convention | Syntax | Meaning | Caller state | After call |
|---|---|---|---|---|
| read | `x: T` (default) | Read-only access; no ownership transfer | Live | Live |
| `inout` | `inout x: T` | Exclusive mutable access | Live | Live (callee must leave it Live) |
| `sink` | `sink x: T` | Ownership transfers into callee | Live | Moved (if not Copy) |
| `set` | `set x: T` | Callee initializes an uninitialized place | Uninit/Moved | Live |

get values are always by value. There are **no reference types in records or get types**; borrows exist only as parameter conventions and scoped bindings (see §10).

---

## 9. Formal grammar

This grammar is **Proposed** for v1.0. It is designed to be LL(1)-friendly except where noted. Conditions are always parenthesized, which removes the record-literal ambiguity that braces-after-expression creates in other languages.

### 9.1 Lexical grammar

```ebnf
(* ---------- Whitespace and comments ---------- *)
whitespace    = " " | "\t" | "\r" | "\n" ;
line_comment  = "//" { any_char_except_newline } ;
block_comment = "/*" { block_comment | any_char } "*/" ;     (* nestable *)

(* ---------- Identifiers ---------- *)
letter        = "A".."Z" | "a".."z" ;
digit         = "0".."9" ;
ident         = ( letter | "_" ) { letter | digit | "_" } ;

(* ---------- Keywords (reserved) ---------- *)
keyword       = "fnc" | "let" | "var" | "record" | "enum" | "type"
              | "const" | "import" | "pub" | "unsafe" | "extern"
              | "if" | "else" | "while" | "loop" | "for" | "in"
              | "match" | "get" | "stop" | "run" | "defer"
              | "true" | "false" | "as" | "sink" | "inout" | "set" ;

(* ---------- Literals ---------- *)
hex_digit     = digit | "a".."f" | "A".."F" ;
dec_int       = digit { digit | "_" } ;
hex_int       = "0x" hex_digit { hex_digit | "_" } ;
oct_int       = "0o" ( "0".."7" ) { "0".."7" | "_" } ;
bin_int       = "0b" ( "0" | "1" ) { "0" | "1" | "_" } ;
int_suffix    = "i8" | "i16" | "i32" | "i64" | "isize"
              | "u8" | "u16" | "u32" | "u64" | "usize" ;
int_lit       = ( dec_int | hex_int | oct_int | bin_int ) [ int_suffix ] ;

exponent      = ( "e" | "E" ) [ "+" | "-" ] dec_int ;
float_lit     = dec_int "." dec_int [ exponent ] [ "f32" | "f64" ]
              | dec_int exponent [ "f32" | "f64" ] ;

escape        = "\\" ( "n" | "r" | "t" | "0" | "\\" | "'" | "\""
                     | "x" hex_digit hex_digit
                     | "u{" hex_digit { hex_digit } "}" ) ;
string_lit    = "\"" { string_char | escape } "\"" ;
char_lit      = "'" ( char_char | escape ) "'" ;
bool_lit      = "true" | "false" ;

(* ---------- Operators and punctuation ---------- *)
arith_op      = "+" | "-" | "*" | "/" | "%" ;
bit_op        = "&" | "|" | "^" | "~" | "<<" | ">>" ;
cmp_op        = "==" | "!=" | "<" | ">" | "<=" | ">=" ;
logic_op      = "&&" | "||" | "!" ;
assign_op     = "=" | "+=" | "-=" | "*=" | "/=" | "%="
              | "&=" | "|=" | "^=" | "<<=" | ">>=" ;
punct         = "(" | ")" | "[" | "]" | "{" | "}"
              | "," | ";" | ":" | "::" | "." | ".." | "..="
              | "::" | "=>" | "?" | "@" ;
```

Lexical rules:

1. Longest match wins (`<<=` before `<<` before `<`).
2. A string literal may not contain a raw newline; use `\n`.
3. `_` alone is a pattern wildcard token, not an identifier.
4. Source files are UTF-8; identifiers are ASCII in v1.0.
5. The generic-closing token `>>` is split into `>` `>` by the parser when closing nested type arguments.

### 9.2 Syntactic grammar

```ebnf
(* ================= Program structure ================= *)
module        = { item } ;

item          = { attribute } [ "pub" ]
                ( fn_item | record_item | enum_item | const_item
                | type_item | import_item | extern_item ) ;

attribute     = "@" ident [ "(" [ attr_arg { "," attr_arg } ] ")" ] ;
attr_arg      = ident | string_lit | int_lit ;

path          = ident { "::" ident } ;

import_item   = "import" path [ "as" ident ] ";" ;

const_item    = "const" ident ":" type "=" expr ";" ;

type_item     = "type" ident [ generic_params ] "=" type ";" ;

generic_params = "<" ident { "," ident } ">" ;

(* ================= Data types ================= *)
record_item   = "record" ident [ generic_params ]
                "{" [ field { "," field } [ "," ] ] "}" ;
field         = [ "pub" ] ident ":" type ;

enum_item     = "enum" ident [ generic_params ]
                "{" variant { "," variant } [ "," ] "}" ;
variant       = ident [ "(" type { "," type } ")" ] ;

(* ================= Functions ================= *)
fn_item       = "fnc" ident [ generic_params ]
                "(" [ param { "," param } [ "," ] ] ")"
                [ "::" type ] ( block | ";" ) ;   (* ";" only for extern / intrinsics *)

param         = [ convention ] ident ":" type ;
convention    = "inout" | "sink" | "set" ;       (* default = read *)

extern_item   = "extern" string_lit "{" { fn_item } "}" ;   (* declarations only *)

(* ================= Types ================= *)
type          = prim_type
              | path [ "<" type { "," type } ">" ]        (* user/generic type *)
              | "(" [ type { "," type } [ "," ] ] ")"     (* tuple / unit *)
              | "[" type ";" expr "]"                     (* fixed array *)
              | "[" type "]"                              (* slice: projection type *)
              | "*" type                                  (* raw pointer: unsafe only *)
              | "fnc" "(" [ type { "," type } ] ")" [ "::" type ]
              | "never" ;

prim_type     = "i8" | "i16" | "i32" | "i64" | "isize"
              | "u8" | "u16" | "u32" | "u64" | "usize"
              | "f32" | "f64" | "bool" | "char" | "str" ;

(* ================= Statements ================= *)
block         = "{" { stmt } "}" ;

stmt          = let_stmt | var_stmt | assign_stmt | expr_stmt
              | if_stmt | while_stmt | loop_stmt | for_stmt | match_stmt
              | get_stmt | stop_stmt | run_stmt | defer_stmt
              | unsafe_block | block ;

let_stmt      = "let" pattern [ ":" type ] "=" expr ";" ;
var_stmt      = "var" ident [ ":" type ] [ "=" expr ] ";" ;
assign_stmt   = place assign_op expr ";" ;
expr_stmt     = expr ";" ;

if_stmt       = "if" "(" expr ")" block
                { "else" "if" "(" expr ")" block }
                [ "else" block ] ;
while_stmt    = "while" "(" expr ")" block ;
loop_stmt     = "loop" block ;
for_stmt      = "for" "(" pattern "in" expr ")" block ;

match_stmt    = "match" "(" expr ")" "{" match_arm { match_arm } "}" ;
match_arm     = pattern [ "if" "(" expr ")" ] "=>" block ;

get_stmt   = "get" [ expr ] ";" ;
stop_stmt    = "stop" ";" ;
run_stmt = "run" ";" ;
defer_stmt    = "defer" ( expr_stmt | block ) ;
unsafe_block  = "unsafe" block ;

(* ================= Patterns ================= *)
pattern       = "_"
              | ident                                      (* binding *)
              | literal
              | path [ "(" [ pattern { "," pattern } ] ")" ]   (* enum variant *)
              | "(" [ pattern { "," pattern } [ "," ] ] ")"    (* tuple *)
              | path "{" [ field_pat { "," field_pat } ] "}"   (* record *)
              | pattern ".." ".=" pattern ;     (* see note on ranges below *)
field_pat     = ident [ ":" pattern ] ;

(* ================= Places ================= *)
place         = ident { "." ident | "[" expr "]" } ;

(* ================= Expressions ================= *)
expr          = or_expr ;

or_expr       = and_expr   { "||" and_expr } ;
and_expr      = cmp_expr   { "&&" cmp_expr } ;
cmp_expr      = bitor_expr [ cmp_op bitor_expr ] ;         (* non-associative *)
bitor_expr    = bitxor_expr { "|" bitxor_expr } ;
bitxor_expr   = bitand_expr { "^" bitand_expr } ;
bitand_expr   = shift_expr  { "&" shift_expr } ;
shift_expr    = add_expr    { ( "<<" | ">>" ) add_expr } ;
add_expr      = mul_expr    { ( "+" | "-" ) mul_expr } ;
mul_expr      = cast_expr   { ( "*" | "/" | "%" ) cast_expr } ;
cast_expr     = unary_expr  { "as" type } ;
unary_expr    = ( "-" | "!" | "~" ) unary_expr
              | postfix_expr ;

postfix_expr  = primary
                { "(" [ arg_list ] ")"            (* call *)
                | "[" expr "]"                    (* index *)
                | "." ident                       (* field or UFCS call target *)
                | "?" } ;                         (* error propagation *)

arg_list      = expr { "," expr } [ "," ] ;

primary       = literal
              | path [ "::" "<" type { "," type } ">" ]    (* explicit instantiation *)
              | "(" expr ")"
              | "(" expr "," [ expr { "," expr } [ "," ] ] ")"   (* tuple *)
              | "[" [ expr { "," expr } [ "," ] ] "]"      (* array literal *)
              | "[" expr ";" expr "]"                      (* repeat literal *)
              | path "{" [ field_init { "," field_init } [ "," ] ] "}"   (* record literal *)
              | "unsafe" block ;                           (* only in unsafe contexts; see §10.9 *)

field_init    = ident [ ":" expr ] ;
literal       = int_lit | float_lit | string_lit | char_lit | bool_lit ;
```

### 9.3 Operator precedence (highest to lowest)

| Level | Operators | Associativity |
|---|---|---|
| 1 | postfix: call `()`, index `[]`, field `.`, `?` | left |
| 2 | prefix: `-` `!` `~` | right |
| 3 | `as` | left |
| 4 | `*` `/` `%` | left |
| 5 | `+` `-` | left |
| 6 | `<<` `>>` | left |
| 7 | `&` | left |
| 8 | `^` | left |
| 9 | `\|` | left |
| 10 | `==` `!=` `<` `>` `<=` `>=` | **non-associative** (`a < b < c` is an error) |
| 11 | `&&` | left |
| 12 | `\|\|` | left |

Bitwise operators bind **tighter than comparisons** (unlike C), so `x & mask == 0` parses as `(x & mask) == 0`.

### 9.4 Grammar notes and known ambiguities

1. **record literal vs block:** an expression statement beginning with `path {` is a record literal; a bare `{` starts a block. Conditions are parenthesized, so `if (x == S { a: 1 }) { ... }` is unambiguous.
2. **Generics in expressions:** `a < b > (c)` would be ambiguous, so type arguments in expressions require `::<`. In *type* positions, `<...>` is unambiguous.
3. **UFCS:** `a.b(c)` is always a UFCS call to a function named `b`. A function-pointer field must be called `(a.b)(c)`.
4. **Range patterns** (`lo ..= hi`) are in the pattern grammar but the exact lexical form is **Open** (§20); the `".." ".="` token pair above is a placeholder for `"..="`.
5. **No expression-bodied blocks in 1.0.** `if` and `match` are statements; values flow through `var` and `get`. (Revisit in 1.x.)
6. **Semicolons** are mandatory statement terminators; a block `}` ends its statement without one.

### 9.5 Attributes (initial set)

| Attribute | Use |
|---|---|
| `@deinit(path)` | On a `record`: the function called to drop it (signature `fnc f(sink x: T)`) |
| `@extern("name")` | Link name for an external function |
| `@intrinsic("name")` | Declares a compiler-provided function |
| `@inline` / `@noinline` | Optimization hints |
| `@noget` | Function never gets (type `never`) |
| `@copy` | Opt a record into `Copy` (only if all fields are `Copy`) |

---

## 10. The memory model (RMM-0)

RMM-0 is the **reference memory model**. It is a *baseline plugin*: the simplest safe model that can write a compiler. Phase 1 may also develop an **original model** (Decision D4, §20), which would be implemented as a second plugin and validated against the same test corpus. RMM-0 is derived from **Mutable Value Semantics** (as in Hylo) and move semantics.

### 10.1 What the model guarantees

| Property | Mechanism | Runtime cost |
|---|---|---|
| No use-after-free | Single owner; use of Moved binding is a compile error | 0 |
| No double-free | Only a Live owner drops; moves transfer the obligation | 0 |
| No dangling stack reference | Borrows are not storable or getable | 0 |
| Iterator/alias invalidation | Exclusivity check at each call | 0 |
| Uninitialized reads | Definite-assignment analysis | 0 |
| Deterministic cleanup | Drops at scope exit and via `defer` | 0 |
| No leaks of owned values (normal paths) | Scope-exit drops | 0 |
| Out-of-bounds | Bounds checks on arrays/`Vec`/slices | 1 branch |
| Integer overflow | Checked arithmetic | 1 branch |
| No null | `Option<T>` | 0 |

Out of scope for the safe model (documented, not hidden): deadlocks, leaks through `unsafe` or process exit, stack overflow (guard page + handler), out-of-memory (panic), correctness of FFI declarations, and data races (v1.0 is single-threaded).

### 10.2 Binding states

Each local binding, at each program point, is in one of:

| State | Meaning |
|---|---|
| `Live` | Holds a valid value; usable |
| `Dead` | Not initialized or moved out (diagnostics distinguish *Uninit* from *Moved*) |
| `Maybe` | Live on some control-flow paths and Dead on others (after a join) |

States are computed by **forward dataflow over RIR-0**, not by structural recursion over the AST. This is what makes loops, early exits, and joins correct.

**Join (⊔):**

| | Live | Dead | Maybe |
|---|---|---|---|
| **Live** | Live | Maybe | Maybe |
| **Dead** | Maybe | Dead | Maybe |
| **Maybe** | Maybe | Maybe | Maybe |

### 10.3 Copy

`Copy(T)` holds when a value can be duplicated bytewise without sharing ownership:

```
Copy(i8..u64, isize, usize, f32, f64, bool, char, ())

Copy(T1) ∧ … ∧ Copy(Tn)            Copy(T)
──────────────────────             ────────────
   Copy((T1,…,Tn))                 Copy([T; n])

all fields Copy ∧ type has @copy   all payloads Copy ∧ type has @copy
──────────────────────────────     ────────────────────────────────
         Copy(record S)                       Copy(enum E)

¬Copy(Box<T>)   ¬Copy(Vec<T>)   ¬Copy(String)   ¬Copy(T) if T has @deinit
```

Copy is **opt-in** for user aggregates (`@copy`) to avoid accidental expensive duplication. `Copy` types have trivial drop.

### 10.4 Operations and state transitions

| Operation | Precondition | Effect |
|---|---|---|
| Read a `Copy` value | `Live` | no change |
| Move a non-Copy value (pass `sink`, `let y = x;`, `get x;`, build aggregate) | `Live` | `Live → Dead (Moved)` |
| Use when `Dead` | n/a | **E1** |
| Use when `Maybe` | n/a | **E4** |
| `x = e;` | any | evaluate `e`; if `Live` drop old; if `Maybe` drop under drop flag; if `Dead` no drop; result `Live` |
| Declare `var x: T;` | n/a | `Dead (Uninit)` |
| `let/var x = e;` | n/a | `Live` |
| Scope exit | n/a | each local declared in the scope that is `Live` is dropped (reverse order); `Maybe` is dropped under its drop flag; `Dead` is not |
| `defer s;` | n/a | `s` runs at scope exit on **every** exit path, LIFO |
| `get` / `stop` / `run` | n/a | drops for all exited scopes run first, in reverse order |

**Important corrections to naïve formulations** (these were bugs in an earlier draft of a similar spec and are called out so they are not reintroduced):

1. After a block, the outgoing state is the block's final state **restricted to variables declared outside it**. A variable moved inside the block stays Moved afterward.
2. Branch joins use ⊔ and **drop flags**, not a requirement that branches leave identical states. `if (c) { consume(s); }` is legal; `s` is `Maybe` afterward and may not be used until reassigned.
3. Loops are solved as a fixed point: the state at the loop head is the join of the entry state and every back-edge state. Moving an outer `Live` variable inside a loop body without reinitializing it makes it `Maybe` at the head, so the use on the next iteration is **E4**.

### 10.5 Parameter conventions: checking rules

For a call `f(a1, …, an)` where parameter *i* has convention *c_i*:

| Convention | At the call | Inside callee | At callee get |
|---|---|---|---|
| read | argument place must be `Live` (or a temporary) | read only; cannot move out of it | unchanged |
| `inout` | argument place must be `Live` | may read and write; may move out of it **only if re-initialized before get** | must be `Live` (else **E6**) |
| `sink` | argument consumed (moved if not Copy) | callee owns; drops it at exit if still Live | n/a |
| `set` | argument place must be `Dead` | callee must initialize it | must be `Live` (else **E6**) |

### 10.6 Exclusivity (the aliasing rule)

Within a single call expression, consider the set of **places** passed to non-Copy-by-value parameters. The rule:

- Any number of **read** borrows of the same place are allowed, **provided no `inout`/`set` borrow of an overlapping place exists in the same call**.
- At most one **`inout`/`set`** borrow may refer to any given place or overlapping place.
- Two places **overlap** when one is a prefix of the other (`x` overlaps `x.a`); sibling fields are disjoint (`x.a` and `x.b` do not overlap); two array elements `a[i]` and `a[j]` are treated as overlapping unless the compiler can prove `i != j`.
- A place borrowed by a call is **frozen** for the duration of the call: it cannot be moved or assigned by another argument expression.

Violations are **E3**. The check is syntactic and local to one call: no lifetimes, no cross-function inference. Because it holds, the LLVM backend may mark `inout` parameters `noalias` and read parameters `readonly` (§13).

### 10.7 Projections and where borrows may appear

Borrows (a read or `inout` view of a place) exist **only** as:

1. parameter conventions (above),
2. scoped pattern bindings in `match`/`let` that destroy a borrowed value,
3. the **projection types** `str` and `[T]`, which may appear as parameter types and as immediately-scoped local `let` bindings.

They **cannot** be stored in record fields, geted from functions, placed in collections, or used as generic type arguments. Attempting any of these is **E5**.

Consequences and workarounds:

- data structure with cross-references use **handles** (`NodeId` = `u32` index into an owning `Vec`/arena). Indexing a handle is bounds-checked.
- A function that needs "an element of a container" either gets it by value (copying or moving it out explicitly) or takes the container by `inout` and mutates in place.
- Compiler tokens store `Span { start: u32, end: u32 }` into the source buffer instead of string slices.

Pattern bindings through `Box<T>` are transparent: matching `Box<Expr>` against `Add(a, b)` binds `a` and `b` as read borrows of the contained `Expr`.

### 10.8 Heap allocation and ownership

- `Box<T>` is an owning pointer to a heap `T`; it is non-Copy, and dropping it drops `T` then frees memory.
- `Vec<T>`, `String`, `Map<K, V>`, and `Arena` are **library types** written in RootCode using `unsafe` and allocator intrinsics (§14).
- **Arenas:** `Arena` owns a block region and hands out typed handles; dropping the arena frees everything at once. This is the intended allocation strategy for compiler passes.
- **Allocation failure** calls the runtime's out-of-memory handler (panic) in 1.0; fallible allocation (`try_alloc`) is deferred.

### 10.9 `unsafe`

`unsafe { ... }` permits exactly: raw pointer dereference and arithmetic, calling `extern` and intrinsic functions that are declared unsafe, `@bitcast`, unchecked indexing, and inline assembly/syscall intrinsics. Two rules:

1. **`unsafe` is a proof obligation on the writer, not a license to skip the rest of the checks.** Moves, drops, bounds checks and overflow checks run to apply inside the block.
2. **Safe code must not be breakable by correct unsafe code.** The standard library's unsafe surface is small, centralized in `lib/alloc` and `lib/rt`, and reviewed as a unit.

By policy, `unsafe` is used only in the runtime and library layers; compiler-proper code is safe RootCode.

### 10.10 Error codes (memory model)

| Code | Name | Meaning |
|---|---|---|
| **E1** | UseOfDead | Variable used after move or before initialization |
| **E2** | MoveOutOfBorrow | Attempt to move out of a read/`inout` parameter or a borrowed binding without re-initialization; or out of a field (partial move, not supported in 1.0) |
| **E3** | ExclusivityViolation | Overlapping `inout`/`set` and another borrow of the same place in one call |
| **E4** | UseOfMaybeDead | Use of a binding that is Live on some paths only |
| **E5** | EscapingBorrow | Projection stored, geted, or used as a type argument |
| **E6** | ConventionNotHonored | `inout`/`set` parameter not `Live` at get |
| **E7** | NonExhaustiveMatch | `match` does not cover all variants |
| **E8** | MissingDeinit | Owning type with a resource but no `@deinit` function |
| **E9** | CopyViolation | `@copy` on a type with a non-Copy component |

Absent by design: lifetime annotations, borrow-region errors, and inference-failure errors; none of those concepts exist in RMM-0.

### 10.11 Worked examples

**Legal: moves and one drop**

```
fnc demo() {
    let a = String::from("hi");   // a: Live
    let b = a;                    // a: Dead(Moved), b: Live
}                                 // drop(b) — exactly one free
```

**Illegal: use after move (E1)**

```
fnc demo() {
    let a = String::from("hi");
    let b = a;
    io::println(a);               // E1: 'a' used after move
}
```

**Legal: conditional move, then reinitialize**

```
fnc demo(flag: bool) {
    var s = String::from("x");
    if (flag) { consume(s); }     // s: Maybe afterward
    s = String::from("y");        // drop under flag, then Live
    io::println(s);               // ok
}                                 // drop(s)
```

**Illegal: use of Maybe (E4)**

```
fnc demo(flag: bool) {
    let s = String::from("x");
    if (flag) { consume(s); }
    io::println(s);               // E4: 's' may have been moved
}
```

**Illegal: exclusivity (E3)**

```
fnc swap2(inout a: Vec<i32>, inout b: Vec<i32>) { ... }

fnc demo() {
    var v = Vec::<i32>::new();
    swap2(v, v);                  // E3: 'v' borrowed twice with inout
}
```

**Legal: disjoint fields**

```
record Pair { a: Vec<i32>, b: Vec<i32> }

fnc demo(inout p: Pair) {
    swap2(p.a, p.b);              // ok: sibling fields do not overlap
}
```

### 10.12 Properties to be proven or tested

These are **targets**, not yet established theorems:

- **Progress/preservation:** a program that passes the checker never executes a use of a Dead binding.
- **No double drop:** along any path, each owned allocation has at most one executed drop.
- **Deterministic destruction:** every `Live` owner is dropped at a statically known point (or a flag-guarded point).
- **Noalias soundness:** under E3, marking `inout` pointers `noalias` is valid for LLVM.

Plan: formalize the rules (a small proof-assistant model is desirable), then back them with property-based and differential testing (§17).

### 10.13 Drop elaboration (RIR-0 → RIR-1)

For each owning binding whose state is `Maybe` at a drop point, the elaborator allocates a **drop flag** (a one-byte local):

```
// source
var s = String::from("x");
if (flag) { consume(s); }
// scope exit

// RIR-1 sketch
%s       = alloca String
%s.flag  = alloca i1
store true  :: %s.flag           ; initialized
cond_br %flag, L_move, L_join
L_move:
  call consume(sink %s)
  store false :: %s.flag         ; moved
  br L_join
L_join:
  %f = load %s.flag
  cond_br %f, L_drop, L_end
L_drop:
  call String_deinit(%s)
  br L_end
L_end:
```

Flags are emitted only for bindings whose state is `Maybe` at some drop point; statically `Live` or `Dead` bindings need none. LLVM's optimizer removes most of the remainder.

---

## 11. Type system

### 11.1 Features (v1.0)

- Nominal records and enums; structural tuples and fixed arrays.
- **Local type inference** for `let`/`var` initializers and generic arguments; **function signatures are always explicit**.
- **Generics are monomorphized and unconstrained:** a generic function or type is type-checked at each instantiation (as in C++ templates and Zig). Error messages for generics therefore report the instantiation site. **Trait bounds are deferred** (§19).
- **No implicit conversions** between numeric types; no implicit `Copy` of non-`@copy` aggregates.
- **Exhaustiveness checking** for `match` on enums, integers (with a wildcard), `bool`, and tuples of these.
- **`never`** is a subtype of every type.
- Recursive types are allowed only through `Box<T>` (or handles).

### 11.2 Algebraic data types

```
enum Option<T> { None, Some(T) }
enum Result<T, E> { Ok(T), Err(E) }
```

Layout: a tag (smallest integer that fits the variant count) followed by a payload region sized for the largest variant, aligned to the strictest payload.

### 11.3 The `?` operator

For an expression of type `Result<T, E>` inside a function geting `Result<U, E2>` where `E2` is `E` (no implicit error conversion in 1.0), `e?` evaluates to the `Ok` payload or performs `get Err(e)` after running pending drops/defers. Likewise for `Option<T>` in a function geting `Option<U>`.

### 11.4 Name mangling (Proposed)

`module::path::name` → `rc_<len>module_<len>path_<len>name`, with instantiations appended as `$<mangled type args>`. The mangling scheme is internal and not an ABI promise.

---

## 12. RIR: the Root Intermediate Representation

RIR is the contract between frontends, memory models, and the backend. It is **typed, CFG-based, and free of memory policy**.

### 12.1 record

- A **module** has globals, type definitions, and functions.
- A **function** has typed parameters (with conventions), locals, and basic blocks.
- A **basic block** is a list of instructions ending in exactly one terminator.
- **Locals** are typed memory slots (not SSA values); LLVM's `mem2reg` promotes them. This keeps the front half simple and keeps drop flags natural.
- Every instruction carries a **source span**.

### 12.2 instruction set (sketch)

**Abstract ownership operations (exist only in RIR-0; consumed by the memory model):**

| Op | Meaning |
|---|---|
| `declare %l : T` | Create a local in state Dead |
| `init %l = value` | Initialize a local |
| `move %dst = %src` | Ownership transfer |
| `copy %dst = %src` | Bitwise duplicate (requires Copy) |
| `drop %l` | Drop at a program point (abstract; elaborated later) |
| `borrow_read %b = place` / `borrow_inout %b = place` | Begin a scoped borrow |
| `end_borrow %b` | End the borrow |
| `scope_enter` / `scope_exit` | Mark scope boundaries (drive scope-exit drops) |
| `defer_push block` | Register a deferred block |

**Concrete operations (RIR-1 and RIR-0):**

| Op | Meaning |
|---|---|
| `const`, `add/sub/mul/div/rem` (+ checked variants), `and/or/xor/shl/shr`, `cmp`, `cast` | Scalar operations |
| `load`, `store`, `field_addr`, `index_addr` | Memory access (index has bounds check variant) |
| `call`, `call_indirect` | Calls |
| `alloc_heap`, `free_heap` | Heap (lowered to runtime allocator calls) |
| `br`, `cond_br`, `switch`, `ret`, `unreachable` | Terminators |
| `intrinsic name(args)` | `@syscall`, `@sizeof`, `@alignof`, `@bitcast`, `@memcpy`, `@trap` |

### 12.3 Verifier

The RIR verifier runs between passes and checks: every block has one terminator; types agree; every use is dominated by an initialization in RIR-1 (after elaboration, no abstract ownership ops remain); every drop flag is initialized on all paths.

---

## 13. LLVM backend and platform layer

### 13.1 Strategy (Decided)

**Emit textual LLVM IR** and invoke `llc` and `lld`. Rationale: no FFI is needed to talk to LLVM, which keeps the self-hosted compiler free of foreign-language bindings, makes output human-readable and testable, and makes a checked-in `.ll` bootstrap seed possible. A later optional step is LLVM's C API or a self-written bitcode writer.

**Version pinning:** LLVM IR syntax changes across releases. RootCompiler pins **one supported LLVM major version** per release and documents it. Exact version is **Open** (§20) and chosen at Phase 1 start.

### 13.2 Type mapping

| RootCode | LLVM |
|---|---|
| `i8…i64`, `u8…u64` | `i8…i64` (signedness is in operations) |
| `bool` | `i1` in registers, `i8` in memory |
| `char` | `i32` |
| `f32`, `f64` | `float`, `double` |
| record | literal record type |
| enum | `{ tag, [N x i8] }` with alignment |
| fixed array `[T; n]` | `[n x T]` |
| `Box<T>` | `ptr` |
| read param of aggregate | `ptr noalias readonly nonnull` |
| `inout` / `set` param | `ptr noalias nonnull` |
| `sink` param of aggregate | passed by value or by hidden pointer, per ABI choice |
| slice `[T]` | `{ ptr, i64 }` |
| function pointer | `ptr` |

The `noalias`/`readonly` attributes are only sound because of the exclusivity rule (§10.6).

### 13.3 Checked arithmetic

```
; RootCode: fnc add(a: i32, b: i32) :: i32 { get a + b; }

declare { i32, i1 } @llvm.sadd.with.overflow.i32(i32, i32)
declare void @rt_panic_overflow() noget

define i32 @add(i32 %a, i32 %b) {
entry:
  %t = call { i32, i1 } @llvm.sadd.with.overflow.i32(i32 %a, i32 %b)
  %v = extractvalue { i32, i1 } %t, 0
  %o = extractvalue { i32, i1 } %t, 1
  br i1 %o, label %trap, label %ok
trap:
  call void @rt_panic_overflow()
  unreachable
ok:
  ret i32 %v
}
```

### 13.4 Platform layer: no libc

The runtime (`lib/rt`, `lib/os`) is written in RootCode and talks directly to the operating system.

| Tier | Targets | Mechanism | Status |
|---|---|---|---|
| **1** | Linux x86_64, Linux aarch64 | Raw syscalls (stable kernel ABI), `_start` provided by `rt` | First target |
| **2** | wasm32 (freestanding/WASI), Linux riscv64, bare metal | Freestanding runtime; WASI imports for wasm | After 0.5 |
| **3** | macOS, Windows | System libraries (libSystem; kernel32/ntdll) because raw syscalls are not a stable interface | Planned |

So "zero dependency" means **no dependency on other programming-language toolchains or runtimes**; Tier 3 targets still depend on the operating system's own interface, which is unavoidable.

### 13.5 Compiler builtins the runtime must provide

LLVM-generated code may call functions normally supplied by `libc` or `compiler-rt`. RootCompiler's `rt` layer must supply them:

- `memcpy`, `memmove`, `memset`, `memcmp`, and `bcmp`
- 128-bit and 64-bit-on-32-bit integer helpers (`__udivti3`, `__umodti3`, `__divdi3`, …) on targets that need them
- Stack probes for large frames (target-dependent)
- Software floating-point helpers on targets without an FPU
- Panic entry points: `rt_panic`, `rt_panic_overflow`, `rt_panic_bounds`, `rt_oom`

### 13.6 Bootstrap and reproducibility

- **Seed:** after 1.0, the stage-1 compiler is checked in as `.ll` for the host target. Building from source needs only LLVM.
- **Cross-targets:** once built, `rcom --target=<triple>` produces other targets; seeds are not needed per target.
- **Hermetic builds:** pinned LLVM version plus the seed yields reproducible builds.
- **Determinism:** the compiler avoids pointer-address ordering and unordered-map iteration in output paths so the fixed-point comparison is meaningful.

### 13.7 Debug info

Phase 1: function and line information only (LLVM `DILocation` from RIR spans). Full variable and type debug info is a 1.x item.

---

## 14. Standard library layers

| Layer | Needs | Contents |
|---|---|---|
| `core` | nothing | Primitives' methods, `Option`, `Result`, integer/float functions, `mem` intrinsics wrappers |
| `alloc` | allocator | `Box`, `Vec`, `String`, `Map`, `Set`, `Arena`, formatting |
| `os` | platform layer | Files, process spawn, environment, time, directories |
| `rt` | target | Entry point (`_start`), panic, compiler builtins, allocator backend (`mmap`/`VirtualAlloc`) |

Library facts:

- The **allocator** sits on page-level OS calls and offers a general allocator plus arenas.
- Hash maps use a deterministic hash (fixed seed) in the compiler build for reproducible output.
- `os::process` exposes one `spawn` function whose implementation is per-platform; the compiler uses it to run `llc` and `lld`.
- Formatting is a library feature (no format-string macros in 1.0).

---

## 15. Phase 3: the language-maker toolkit

> **This entire section is a sketch.** It describes the intended shape, not a stable interface. The first concrete design will come only after Phase 2, informed by what the self-hosted compiler needed.

### 15.1 Goal

A user can define a new language by supplying four things, and RootCompiler supplies everything else (parsing engine, passes, RIR, LLVM backend, diagnostics):

| User supplies | Role |
|---|---|
| **Syntax** | A grammar definition producing an AST |
| **Semantics** | Name resolution and type rules (as hooks or declarative rules) |
| **Memory model** | A plugin that analyzes and elaborates RIR-0 ownership operations |
| **Lowering** | AST/HIR → RIR |

### 15.2 Extension points (illustrative)

```
interface Frontend {
    grammar   : Grammar            // syntax definition
    resolve   : (Ast) :: Resolved
    typecheck : (Resolved) :: Typed
    lower     : (Typed) :: RirModule      // must use RIR-0 ownership ops
}

interface MemoryModel {
    name       : string
    analyze    : (RirFunction, Diagnostics) :: ()    // enforce rules, emit errors
    elaborate  : (RirFunction) :: ()                 // lower abstract ops to concrete RIR-1
    layout     : (Type) :: LayoutHints               // e.g. extra header words
}

interface Pass {
    name  : string
    runs_after, runs_before : [string]
    run   : (RirModule) :: RirModule
}
```

### 15.3 Grammar definition language (illustrative)

```
grammar Calc {
    expr   = term { ("+" | "-") term } ;
    term   = factor { ("*" | "/") factor } ;
    factor = NUMBER | "(" expr ")" ;
}
```

Intended derived artifacts: parser (PEG or LL-style), AST node types, visitors, pretty-printer/formatter, and diagnostic locations.

### 15.4 What "bring your own memory model" means

A memory-model plugin receives RIR-0 containing abstract operations (`move`, `copy`, `drop`, `borrow_*`, scope markers) and decides:

- **which rules to check** (e.g. RMM-0's states and exclusivity; a GC model might check nothing),
- **what concrete code to emit** (explicit drops; reference-count increments; generation checks; or calls into a collector),
- **what layout changes** it needs (e.g. an extra header word for reference counts).

Examples this should be able to express: RMM-0 (the reference), a reference-counting model, a tracing-GC model whose collector is library code, a pure-arena model, and an original model developed as part of Decision D4.

### 15.5 Acceptance test for Phase 3

> A second, **structurally different** language (different syntax *and* different memory model, for example a small Lisp-like language with a garbage collector) is built using **only the public extension mechanism**, in days rather than months, and RootCode itself is re-expressed through the same mechanism.

Until this test passes, the "mother of languages" claim is unproven and must be labeled as such.

### 15.6 Hard problems to expect

- **Diagnostics for user-defined languages:** good error messages for a language you did not hand-write are difficult; spans and record diagnostics must be in RIR from the start.
- **Safe extensibility:** a buggy memory-model plugin can produce unsound code. A RIR verifier and a conformance test kit for plugins are required.
- **Stable plugin interfaces** in a self-hosted, evolving language.
- **Tooling:** editor support (syntax highlighting, language server) for user languages.

---

## 16. Diagnostics

- Every AST node and RIR instruction carries a **source span** (file id, start, end).
- Diagnostics have: **code** (e.g. `E1`), **severity**, **primary span**, **secondary spans** (e.g. "moved here"), **message**, and optional **help**.
- The parser **recovers** at `;` and `}` and runs, so several errors are reported per run.
- `rcom --explain E1` prints a long-form explanation and example.
- Output is stable and machine-readable (`--format=json`) for tooling and tests.

Example:

```
error[E1]: use of moved value 'a'
  --> demo.root:4:15
   |
 2 |     let a = String::from("hi");
   |         - 'a' declared here
 3 |     let b = a;
   |             - value moved here
 4 |     io::println(a);
   |                 ^ value used here after move
```

---

## 17. Testing and verification strategy

| Layer | Method |
|---|---|
| Grammar | A single machine-readable grammar file; parser tests and docs generated from it so they cannot drift |
| Lexer/parser | Golden-file tests plus **fuzzing** (crash-free, bounded time) |
| Typechecker | Positive/negative corpus with expected error codes and spans |
| Memory model | Conformance suite: one test per rule and per error code; **property-based tests** on random RIR for "no double drop / no use of dead" |
| Codegen | Run-and-compare tests on Tier 1 targets; `.ll` golden files for small programs |
| Differential | Same program compiled in debug/release (checks must agree); D-hosted vs self-hosted outputs |
| Self-hosting | **Fixed-point test:** stage1 compiles the compiler into stage2; stage2 compiles it into stage3; **stage2 and stage3 must be byte-for-byte identical** |
| Spec drift | CI fails when a documented example no longer parses or checks as documented |

Bootstrap stages:

| Stage | Compiler | Built by |
|---|---|---|
| stage0 | D implementation | D toolchain |
| stage1 | RootCode implementation | stage0 |
| stage2 | RootCode implementation | stage1 |
| stage3 | RootCode implementation | stage2 (must equal stage2) |

Rule: **stage1's source may use only features that stage0 implements.** Phase 3 features are never required to self-host.

---

## 18. Dependency policy

| Dependency | Phase 1 | After 1.0 | Policy |
|---|---|---|---|
| D compiler (LDC) and runtime | yes (used `@nogc`, no Phobos) | **no** | Retired by the seed |
| C compiler, libc | **no** | **no** | Never required |
| System linker/assembler | no (`lld` from LLVM) | no | |
| Other languages (Python, shell scripts for build) | avoid | avoid | Build is driven by `rcom` itself |
| **LLVM (`llc`, `lld`, headers for IR format)** | **yes** | **yes** | **Sole permitted dependency, at least through stable 2.0 (Decided)** |
| OS interface | yes | yes | Unavoidable; thin platform layer |

**Decided:** LLVM stays the only dependency at least until the stable 2.0 release. Whether to reduce reliance on LLVM later (own object writer, linker, or alternative backends) will be decided **only after RootCompiler has achieved broad adoption**; until then it is explicitly out of scope.

---

## 19. Limitations and risks

### 19.1 Language limitations (planned for 1.0)

- **No closures**, so no first-class capturing functions; use function pointers and explicit environment records.
- **No threads, atomics, or async.** The safe model is single-threaded; a concurrency model (e.g. `Send`-style marking) is a post-1.0 design item.
- **No traits/interfaces:** generics are unconstrained and checked at instantiation, which means worse generic error messages and no bounded polymorphism.
- **No compile-time evaluation (`comptime`) or macros.** Code generation is via the Phase 3 toolkit instead.
- **No references in data structures:** programs that need pervasive sharing (graphs, doubly linked lists, observer patterns) must use handles/indices.
- **No partial moves** out of record fields in 1.0.
- **No unwinding:** panics terminate the process.
- **Value semantics have a copy cost**; mitigated by moves, projections, and `@copy` opt-in, but real.
- **FFI is limited** to `extern` declarations with C-compatible types.

### 19.2 Project and technical risks

| Risk | Impact | Mitigation |
|---|---|---|
| **Scope:** compiler, language, memory model, workbench, platform layer at once | Project stalls | Phased plan, kill criterion (§6.3), strict self-hosting scope |
| **Original memory model is a research problem** | Unsound or unusable model | Ship RMM-0 first; treat the original model as a plugin validated by the conformance suite |
| **Phase 3 is research-grade** | "Mother of languages" remains aspirational | Acceptance test (§15.5); defer promises until proven |
| **LLVM IR drift** | breakage across LLVM releases | Pin a version per release; keep IR emission in one module |
| **Self-hosted compiler needs features that arrive late** | Self-hosting delayed | Write the compiler to a deliberately small feature set; develop the lexer/parser in RootCode first as a probe |
| **Missing runtime pieces** (`memcpy`, 128-bit math, stack probes) | Link failures | Explicit builtin list (§13.5) and early cross-target tests |
| **macOS/Windows have no stable raw-syscall interface** | Zero-libc is Linux/wasm-only | Tiered platform strategy (§13.4) |
| **Diagnostics for user languages** | Poor developer experience | Spans and structured errors from day one |
| **Plugin unsoundness** | Memory-model plugins can stop safety | RIR verifier; plugin conformance kit |
| **Single maintainer** | Bus factor | Small codebase, documentation-first, spec tests |
| **Soundness unproven** | Hidden bugs in RMM-0 | Formalization plus property-based testing (§10.12) |

### 19.3 What this project does not claim

- It does **not** claim a new optimizer or better code quality than existing LLVM-based languages.
- It does **not** claim memory safety beyond the documented model (§10.1) and outside `unsafe`.
- It does **not** claim the Phase 3 toolkit works until the acceptance test passes.
- It does **not** claim independence from LLVM; LLVM is a declared, deliberate dependency.

---

## 20. Decision register and open questions

### 20.1 Decided

| ID | Decision |
|---|---|
| D1 | Project name **RootCompiler**, CLI **`rcom`**, language **RootCode** |
| D2 | Phase 1 in **D without GC**; **LLVM IR from day one** |
| D3 | Phase 2: **self-host** RootCompiler in RootCode (D retired afterward) |
| D4 | Pursue an **original memory model**, distinct from existing languages, as a plugin; RMM-0 is the baseline until then |
| D5 | Phase 3: make RootCompiler a toolkit so anyone can build a language with their own **syntax, style, features, and memory model** |
| D6 | **LLVM is the only dependency** through at least stable **2.0**; revisit after popularity |
| D7 | Self-hosting is the acceptance test; if scope slips, cut features, not the goal |
| D8 | Target near-zero dependency on other programming languages and run on **any LLVM-supported device** (tiered, §13.4) |

### 20.2 Proposed (current best guess)

| ID | Proposal |
|---|---|
| P1 | Textual LLVM IR plus `llc`/`lld` as the backend interface |
| P2 | Mutable Value Semantics as the basis of RMM-0, with parameter conventions `read/inout/sink/set` |
| P3 | Parenthesized conditions; `()` calls, `[]` indexing/array types, `{}` blocks; UFCS method calls |
| P4 | `fnc`, `let`, `var`, `get`, `::` for result types (RasCode's `get[...]`/`::` retired) |
| P5 | Monomorphized, unconstrained generics in 1.0 |
| P6 | Checked integer arithmetic by default in all build modes |
| P7 | Handles/indices instead of stored references; projection types `str`, `[T]` as parameter-only |
| P8 | Release numbering: 1.0 = self-hosted, 2.0 = stable language-maker toolkit |

### 20.3 Open questions

| ID | Question |
|---|---|
| O1 | **What is the core insight of the original memory model?** (The problem it solves, its guarantees and its costs.) Needed before it can be specified. |
| O2 | Final source file extension (`.root` is a placeholder) |
| O3 | Exact pinned LLVM major version |
| O4 | Range pattern syntax and whether `match` should be an expression in 1.0 |
| O5 | Whether to keep RasCode's bracket-call style as an *alternative frontend* once Phase 3 exists |
| O6 | Slice/`str` projection typing details (can they be bound in `var`?) |
| O7 | Trait system design and timing (1.x vs 2.0) |
| O8 | Concurrency model (post-1.0) |
| O9 | License and governance |
| O10 | Package manager and distribution (post-1.0) |

---

## 21. Example programs

> Examples illustrate the **Proposed** syntax. They are not yet accepted by any compiler.

### 21.1 Hello world

```
import os::io;

fnc main() :: i32 {
    io::println("Hello, RootCode!");
    get 0;
}
```

### 21.2 Recursion and checked arithmetic

```
fnc factorial(n: u64) :: u64 {
    if (n <= 1) { get 1; }
    get n * factorial(n - 1);      // traps on overflow
}
```

### 21.3 Sum types and `match`

```
enum Expr {
    Num(i64),
    Add(Box<Expr>, Box<Expr>),
    Neg(Box<Expr>),
}

fnc eval(e: Expr) :: i64 {            // read convention: borrowed, not consumed
    match (e) {
        Num(n)    => { get n; }
        Add(a, b) => { get eval(a) + eval(b); }   // a, b are borrowed through Box
        Neg(x)    => { get 0 - eval(x); }
    }
}
```

### 21.4 Error handling with `?`

```
enum ParseError { BadChar(u8), Empty }

fnc digit_value(c: u8) :: Result<u8, ParseError> {
    if (c >= 48 && c <= 57) { get Ok(c - 48); }
    get Err(ParseError::BadChar(c));
}

fnc parse_two(a: u8, b: u8) :: Result<u8, ParseError> {
    let x = digit_value(a)?;
    let y = digit_value(b)?;
    get Ok(x * 10 + y);
}
```

### 21.5 Parameter conventions

```
fnc push_twice(inout v: Vec<i32>, x: i32) {
    v.push(x);                        // UFCS: push(v, x), where push has an inout first parameter
    v.push(x);
}

fnc take_and_print(sink s: String) {
    io::println(s);
}                                      // s dropped here

fnc main() :: i32 {
    var v = Vec::<i32>::new();
    push_twice(v, 7);
    let s = String::from("bye");
    take_and_print(s);                 // s moved
    // io::println(s);                 // would be E1
    get 0;
}
```

### 21.6 `defer` and cleanup order

```
fnc work() :: Result<(), IoError> {
    let f = os::open("data.txt")?;
    defer os::close(f);                // runs on every exit path
    let n = os::read_all(f)?;
    io::println(n);
    get Ok(());
}
```

### 21.7 Handles instead of references (compiler-style data)

```
record NodeId { index: u32 }           // @copy: a small integer
record Span   { start: u32, end: u32 } // @copy

enum Node {
    Number(Span),
    Binary(NodeId, u8, NodeId),
}

record Ast { nodes: Vec<Node> }

fnc add_node(inout ast: Ast, sink n: Node) :: NodeId {
    let id = NodeId { index: ast.nodes.len() as u32 };
    ast.nodes.push(n);
    get id;
}
```

(`@copy` attributes omitted above for brevity: `NodeId` and `Span` would carry `@copy`.)

### 21.8 Unsafe runtime code (library layer only)

```
@intrinsic("syscall3")
fnc syscall3(n: usize, a: usize, b: usize, c: usize) :: isize;

pub fnc write(fd: i32, buf: str) :: isize {
    unsafe {
        get syscall3(1, fd as usize, buf.ptr() as usize, buf.len());
    }
}
```

---

## 22. Appendices

### A. Glossary

| Term | Meaning |
|---|---|
| **RootCompiler** | The overall project and toolkit |
| **`rcom`** | The command-line compiler driver |
| **RootCode** | The reference language |
| **RIR** | Root Intermediate Representation (typed, CFG-based, policy-free) |
| **RMM-0** | Baseline memory model (mutable value semantics, moves, deterministic drop) |
| **Plugin** | A semantic module (type rules, memory model, pass) loaded into the pipeline |
| **Self-hosting** | The compiler is written in the language it compiles |
| **Fixed point** | stage2 and stage3 compiler outputs are identical |
| **Seed** | A checked-in LLVM IR file used to bootstrap without the original implementation language |
| **MVS** | Mutable Value Semantics: values with single owners and no shared mutable aliasing |
| **Projection** | A scoped, non-storable borrow of a place |
| **Handle** | A small typed index into an owning container used in place of a pointer |
| **Drop flag** | A runtime boolean recording whether a conditionally moved value still needs dropping |
| **Monomorphization** | Instantiating generic code separately for each concrete type argument |

### B. RasCode → RootCode migration

| RasCode | RootCode | Reason |
|---|---|---|
| `fnc name[params]::type { }` | `fnc name(params) :: type { }` | One bracket, one job |
| `get[value];` | `get value;` | Conventional, unambiguous; `get` was also a map operation |
| `if[c] { } or [c2] { } or { }` | `if (c) { } else if (c2) { } else { }` | Familiar; no keyword overloading of `or` |
| `arr{int, 5} a = {…};` | `let a: [i32; 5] = [ … ];` | `{}` reserved for blocks and literals |
| `name{index}` | `name[index]` | Indexing uses `[]` |
| `func[args]` | `func(args)` | Calls use `()` |
| `deci`, `byte`, `ubyte`, `none` | `f64`, `i8`, `u8`, `()` | Explicit widths for LLVM |
| `map{k, v} m;` + `m::set[k][v]` | `Map<K, V>` + `m.insert(k, v)` | Library type; UFCS |
| `check { } or { }` | `Result<T, E>` + `?` | Typed errors |
| `manme { }` | `Arena` library type + RMM-0 | Real allocator, explicit handles |
| `showf["...$x..."]` | library formatting | No built-in interpolation in 1.0 |

### C. Known RasCode bugs this design addresses

| RasCode issue | Resolution |
|---|---|
| Brackets overloaded across call, condition, index, type | Grammar redesigned (§9) |
| Spec and examples disagreed (comment style, operators) | Single grammar file, doc examples tested in CI (§17) |
| Full C-style `loop` header segfaulted in codegen | All loops lower to one CFG form in RIR (§12); one lowering to test |
| `manme` had escape analysis but no allocator | Arena is a real library type; escape rule is E5 (§10.7) |
| Docs outdated relative to parser | Grammar generated from the tested source of truth |

### D. Reading list

**Memory models:** Hylo / Val documentation on Mutable Value Semantics; Vale's hybrid generational references; Cyclone and MLKit region papers; Pony reference capabilities; Perceus (Koka); *The Rust Programming Language*, chapters on ownership and lifetimes.

**Compilers:** *Crafting Interpreters* (Nystrom); *Engineering a Compiler* (Cooper & Torczon); *Modern Compiler Implementation* (Appel); the nanopass framework papers; the LLVM Language Reference and *Kaleidoscope* tutorial; the MLIR documentation (dialects).

**Language workbenches:** Racket's `#lang` documentation; JetBrains MPS; Spoofax; Rascal.

**Systems:** *The Linux Programming Interface* (Kerrisk); *Computer Systems: A Programmer's Perspective*.

### E. First milestone checklist (Phase 1, week 1)

1. Install LDC and LLVM (`llc`, `lld`); pin the LLVM version (O3).
2. Create the repository layout (§7.5); commit this document and `DECISIONS.md` (§20).
3. Write the lexer in D (`@nogc`, own arena) with golden tests.
4. Parse the §21.1 and §21.2 programs to an AST and pretty-print them.
5. Hand-write the target `.ll` for §21.1 and §21.2 and confirm `llc` + `lld` produce a running binary on Linux x86_64 using the raw-syscall `rt`.
6. Only then begin the typechecker and RIR.

*End of document.*

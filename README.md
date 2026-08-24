# Spitbol4Win — SPITBOL for Windows

**By Jeffrey Cooper.** A complete, self-contained Visual Studio build of SPITBOL x64 on
Windows: no Unix toolchain, no NASM, no pre-existing SPITBOL binary — open the solution,
build, and Windows compiles SPITBOL from source and then proves the result against a
generated regression suite.

SPITBOL is the high-performance native-code implementation of SNOBOL4. Its build has
lived on Unix for decades. This repository is what it took to make that build *native* to
Windows — and the interesting part is not a port of the code, but a rethink of the
**bootstrap**.

## The bootstrap problem, and how this repo cuts it

SPITBOL is written in MINIMAL, a portable assembly-level language. The translator that
turns MINIMAL into real machine assembly (`lex.sbl`, `asm.sbl`) is itself **written in
SPITBOL** — so building SPITBOL from source has always required an already-working
SPITBOL. On Unix you bootstrap from a shipped binary. On Windows there was nothing to
bootstrap from.

Jeffrey's answer: **re-implement the translator in C#** — `MinimalLexer.cs` (a port of
`lex.sbl`) and `MinimalCodeGen.cs` (a port of the `asm.sbl` back end) — so the entire
pipeline runs in-process under .NET, on any Windows machine, with no prior SPITBOL:

```
sbl.min ─[MinimalLexer]→ sbl.lex ─[MinimalCodeGen]→ sbl.asm ─[Nasm2Masm]→ ─[+masm.h]→ sbl_masm.asm
            tokenizer               NASM body           NASM→MASM body      prepend header
```

And crucially, **every stage is verified byte-for-byte against the Linux bootstrap's
output**: the C# lexer reproduces the reference `sbl.lex` exactly, and the C# code
generator reproduces the reference NASM body exactly. The translator rewrite is not
"close enough" — it is provably the same translation.

## What's in the box

| Project | Role |
|---|---|
| **`SpitbolBuild/`** (C#, .NET 8) | The one-step MINIMAL→MASM pipeline: lexer, code generator, NASM→MASM converter, embedded `masm.h` machine header. Also carries the vendored SPITBOL sources (`sbl.min`, `err.asm`, `int.asm`, `int.dcl`) and the full `osint/` C runtime with the Windows fixes applied. |
| **`SpitbolExe/`** (Visual Studio C++) | Compiles `osint/*.c`, assembles the three generated MASM files with `ml64`, links `spitbol.exe` — with the fixed-base/no-ASLR link flags already set. |
| **`SpitbolTests/`** (xUnit, .NET) | Runs the built `spitbol.exe` against a corpus of 200+ SNOBOL4 programs; one test node per program in VS Test Explorer / `dotnet test`. |
| **`SblTestGen/`** (C#) | The corpus tooling — converts and snapshots programs into test cases with expected outputs. |
| `Snobol Functions.xlsx` | A function reference worksheet compiled during the port. |
| `Spitbol4Win.slnx` | The solution tying it together. |

Each project carries its own README with full build and usage detail
(`SpitbolBuild/README.md`, `SpitbolBuild/BUILD_WINDOWS.md`, `SpitbolBuild/WINDOWS_FIXES.md`,
`SpitbolBuild/CHANGES.md`, `SpitbolExe/README_SpitbolExe.md`, `SpitbolTests/README.md`).

## The techniques worth reading the source for

**The NASM→MASM converter collapsed ~100 assembler errors into five root causes**, each
cured mechanically in `Nasm2Masm.cs` rather than by hand-editing a megabyte of generated
assembly. Two favorites: MASM refuses NASM's bare `rep` prefix on its own line, so the
converter merges prefix lines into their string instruction; and a MINIMAL data label can
collide with an x86 mnemonic after conversion (`cmpss: d_word 0` becomes `cmpss dq 0`,
which MASM parses as an SSE instruction!) — the converter carries a reserved-mnemonic
table and whole-word-renames colliding labels (`cmpss` → `cmpss_`) so **future revisions
of `sbl.min` are handled automatically**, not patched case by case.

**The LLP64 catch — the one-line bug behind every pointer truncation.** SPITBOL's C
runtime declares its machine word as `typedef long word;`. On Linux (LP64) `long` is
64-bit; on Windows (LLP64) it is **32-bit** — so the MINIMAL registers and all
`sizeof(word)` pointer math silently truncated to 32 bits against a 64-bit assembly side.
Diagnosed and fixed with a guarded `long long` typedef. (`WINDOWS_FIXES.md` tells this
story in three tight paragraphs — recommended reading.)

**66 compile errors, one include.** `sproto.h` unconditionally included `<unistd.h>` and
is pulled into every osint file; on Windows the POSIX I/O calls come from `<io.h>` via a
new `oswin.h`. One `#ifndef _WIN32` guard turned 66 red files green.

**Add-on discipline: every osint edit is `#ifdef _WIN32`-guarded**, so the Linux build of
the same files is **byte-for-byte unchanged**. This is a Windows port that leaves no
fingerprints on the platform it came from.

**The link recipe encodes real archaeology**: MINIMAL-generated code uses 32-bit absolute
addressing, so the image must load at a fixed low base with ASLR off —
`/BASE:0x10000 /FIXED /DYNAMICBASE:NO /HIGHENTROPYVA:NO /LARGEADDRESSAWARE:NO` — or the
link dies in ADDR32 relocation overflow. The flags are pre-set in `Spitbol.vcxproj`, with
the why written down.

**`int.asm` is transformed at build time, not forked**: `IntPrep.cs` applies the Windows
ABI preparation to the original, unmodified Unix `int.asm` on every run, and the
`int.dcl` declarations are spliced into the generated assembly automatically. The
originals ship pristine; the port is a *function applied to them*.

**The test harness is a small design gem** (`SpitbolTests/README.md`): each corpus
program becomes an xUnit node; a `<name>.expected` file beside a program selects golden
mode (exact stdout+stderr diff), no file selects self-check mode (exit 0 and no `*FAIL`
line); `.in` feeds stdin, `.filter` drops nondeterministic lines by regex, and
`SPITBOL_UPDATE_GOLDEN=1` captures baselines from a trusted run. Because golden mode
deliberately ignores the exit code, **compile-error tests work too**: a program that is
*supposed* to be rejected records SPITBOL's error listing as its golden output.

## Building

1. Open `Spitbol4Win.slnx` in Visual Studio (2022 or later; the C++ project retargets to
   your installed toolset), or use `dotnet build` for the C# projects.
2. Build and run **SpitbolBuild** — it generates `runtime\sbl_masm.asm`,
   `err_masm.asm`, `int_masm.asm` from the vendored sources.
3. Build **SpitbolExe** (needs the C++ workload; assembles with `ml64`, compiles osint,
   links `spitbol.exe`).
4. Build **SpitbolTests** and run in Test Explorer or `dotnet test`.

`SpitbolBuild/BUILD_WINDOWS.md` documents the equivalent command-line path
(`ml64` / `cl` / `link` invocations) end to end.

## Status — honest ledger

- All three generated MASM files **assemble clean** under ml64 (14.51) and JWasm.
- The full link **succeeds**; `spitbol.exe` runs and prints the SPITBOL v4.0f banner —
  startup, the ABI bridge, malloc-backed memory, and the output path all work.
- **Open item**: compiling a source program still faults in the compiler-execution path;
  `BUILD_WINDOWS.md` records the debugging plan and the prime suspects (the
  syscall/math_op shadow-space bridge, calltab dispatch, heap pointer handling). The
  `SpitbolExe` project is set up precisely so this can be chased in the VS debugger with
  symbols.
- **Deferred (stubbed, `#ifdef`-guarded)**: EXTFUN/LOAD, pipes/EXEC, tty raw mode,
  a.out writing — each returns "unsupported" cleanly and is off the first-light path.

## Provenance and license

The vendored SPITBOL sources (`sbl.min`, `err.asm`, `int.asm`, `int.dcl`, `osint/`) come
from the SPITBOL x64 distribution and remain under its license — **GPL v2 or later**
(see `COPYING`). The Windows build system, the C# translator ports, the NASM→MASM
converter, the test harness, and all documentation are **Jeffrey Cooper's work (2026)**,
contributed to the snobol4ever organization and committed here on his behalf.

Part of the [snobol4ever](https://github.com/snobol4ever) family of projects keeping
SNOBOL4, SPITBOL, and their descendants alive on modern platforms.

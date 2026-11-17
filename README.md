# Universal C

Rules for writing C that **compiles on every conforming compiler from C89/C90 through C23**
and **behaves the same on every mainstream platform**.

The rules live in a single file, [`UNIVERSAL_C.md`](UNIVERSAL_C.md). It is written as
instructions for an LLM coding assistant, but it reads just as well as a style guide for
people.

## What "universal" means here

- **Compiles** with zero warnings under `gcc` and `clang` with
  `-std={c89,c99,c11,c17,c2x} -Wall -Wextra -pedantic-errors -Werror`.
- **Behaves identically** on any hosted implementation with 8-bit bytes: 16/32/64-bit,
  little- or big-endian, signed or unsigned plain `char`, LP64 (Linux/macOS) or LLP64
  (Windows).
- **No undefined behavior and no reliance on implementation-defined behavior.**
- C++ compatibility is opt-in (§10).

In practice this means writing in the C89 dialect, using only the C89 standard library,
avoiding every keyword and reserved name added by later standards, and routing all
compiler- and version-specific logic through one small portability header.

## What's in `UNIVERSAL_C.md`

| § | Topic |
|---|---|
| 0 | Target definition |
| 1 | Syntax: the C89 dialect (no `//`, no mixed declarations, no trigraphs, …) |
| 2 | Features later standards removed (implicit `int`, K&R definitions, `gets`) |
| 3 | Keywords and reserved identifiers across C99, C11, C23 and C++ |
| 4 | `port.h`: a drop-in portability header (exact-width integers, `P_INLINE`, static assert, 64-bit detection) |
| 5 | Standard library allow-list, plus `p_util.c` helpers (overflow-checked arithmetic, bounded copy, endian-independent serialization, 64-bit decimal formatting) |
| 6 | Undefined behavior to avoid |
| 7 | Implementation-defined behavior not to depend on |
| 8 | Floating point |
| 9 | C89 translation limits |
| 10 | C++ compatibility (opt-in) |
| 11 | Isolating code that cannot be universal (threads, sockets, …) |
| 12 | Checklist to run before outputting code |
| 13 | Verification commands: multi-standard builds, sanitizers, 32-bit, Windows, big-endian, static analysis |

## Usage

**With an LLM coding assistant:** add `UNIVERSAL_C.md` to the assistant's context, for
example by referencing it from your project's instruction file (such as `AGENTS.md`),
pasting it into a system prompt, or attaching it to a conversation. Then ask for
"universal C". The document tells the model to follow every **MUST**/**MUST NOT** rule,
run the §12 checklist, and say plainly when part of a request cannot be done portably.

**By hand:** copy `port.h` and `p_util.h`/`p_util.c` out of §4 and §5 into your project,
follow the rules, and run the §13 verification script in CI.

## Verifying your code

§13 contains a script that builds under every standard with gcc and clang, runs the
tests under AddressSanitizer, UndefinedBehaviorSanitizer and MemorySanitizer, and
cross-builds for unsigned `char`, 32-bit x86, 64-bit Windows (MinGW + Wine) and big-endian
s390x (QEMU). The same inputs should give the same output on every configuration;
for text streams, compare after normalizing line endings.

## Scope and limits

- Requires `CHAR_BIT == 8`; a hosted environment is assumed unless you say otherwise.
- A 64-bit integer type is optional in C89; code must check `P_HAS_64`.
- Threads, atomics, sockets, filesystems, SIMD and high-resolution timers cannot be
  universal. §11 describes how to keep them behind small per-platform interfaces.
- Bit-identical floating-point results across platforms are not achievable; use integer
  or fixed-point arithmetic when results must match exactly.
- For high-assurance code, also consult SEI CERT C and MISRA C.

# Universal C: Rules for Writing Maximally Portable C

> **Instructions for the LLM reading this file.** The user wants "universal C": code that
> **compiles** on every conforming C compiler from C89/C90 through C23, and that **behaves
> identically** on every platform. Follow every rule marked **MUST**/**MUST NOT** below
> without exception, unless the user explicitly overrides a specific rule. Before
> outputting any code, run the checklist in §12. If the user asks for something that cannot
> be done universally, say so plainly and isolate the non-portable part (see §11). Do not
> silently fall back to C99+ features or compiler extensions.

---

## 0. Target definition

- **Compile target:** the common subset of ISO C90, C95, C99, C11, C17 and C23.
  The code MUST compile with **zero warnings** under all of:
  `gcc`/`clang -std={c89,c99,c11,c17,c2x} -Wall -Wextra -pedantic-errors -Werror`.
- **Behavior target:** identical observable output on any hosted implementation where
  `CHAR_BIT == 8`: 16/32/64-bit, little-/big-endian, signed or unsigned plain `char`,
  LP64 (Linux/macOS) or LLP64 (Windows).
- **C++ compatibility** is **off by default**. Apply §10 only if the user asks for code that
  also compiles as C++.
- **Hosted vs. freestanding:** assume hosted (full C89 library). If the user says
  freestanding/bare-metal, only `<float.h> <limits.h> <stdarg.h> <stddef.h>` are guaranteed.

---

## 1. Syntax: write in the C89 dialect

| MUST NOT use | Use instead |
|---|---|
| `// comments` | `/* comments */` |
| Declarations after statements in a block | Put all declarations at the top of a `{ }` block (open a new block if needed) |
| `for (int i = 0; ...)` | Declare `i` at block top |
| Trailing comma in `enum` list | No comma after the last enumerator |
| Designated initializers `{ .x = 1 }`, `[3] = 1` | Positional initializers in declaration order, or assign members after declaration |
| Compound literals `(struct p){1, 2}` | Named temporary variable |
| Variable-length arrays `int a[n]` | `malloc` or a fixed maximum size |
| Flexible array members `char data[];` | Separate allocation, or a fixed-size array |
| Variadic macros `...`, `__VA_ARGS__` | Variadic *functions* with `<stdarg.h>`, or several fixed-arity macros |
| `long long`, `LL`/`ULL` suffixes | `p_u64`/`p_s64` from §3, guarded by `P_HAS_64` |
| `_Bool`, `bool`, `<stdbool.h>` | `p_bool`, `P_TRUE`, `P_FALSE` |
| `inline` / `restrict` directly | `P_INLINE` / `P_RESTRICT` |
| `__func__` | `__FILE__` / `__LINE__`, or a string literal |
| `_Static_assert` / `static_assert` | `P_STATIC_ASSERT(cond, tag)` at file scope |
| Empty macro arguments `F(,)` | Explicit arguments |
| Mixed-type `?:` tricks relying on `_Generic` | Separate functions per type |
| An empty translation unit | Every `.c` file must contain at least one declaration |
| `??` followed by `= ( / ) ' < ! > -` anywhere (trigraphs) | C89–C17 replace these; C23 removed trigraphs, so `"what??!"` prints `what\|` or `what??!` depending on `-std`. Write `"?" "?!"` or `\?\?` |
| Digraphs `<: :> <% %> %: %:%:` | The real tokens (digraphs are C95) |
| Hex float constants `0x1p3`, universal character names `é` | Decimal constants; UTF-8 bytes written as `\xNN` escapes |
| `[static N]` / qualifiers inside array parameter brackets | Plain pointer parameter |
| Anonymous struct/union members | Named members |

Non-initializer expressions in aggregate initializers of automatic variables
(`int a[2] = { x, y };` where `x` is a variable) are C99-only. Initializers of arrays,
structs and unions MUST be constant expressions; assign runtime values afterward.

---

## 2. Things later standards removed: never use them

- **MUST** give every function an explicit return type (no implicit `int`).
- **MUST** declare (prototype) every function before calling it. Include the right header.
- **MUST** write `(void)` for an empty parameter list: `int f(void);` never `int f();`.
- **MUST NOT** use K&R-style definitions (`int f(a) int a; { }`). C23 removed them.
- **MUST** define `main` as `int main(void)` or `int main(int argc, char **argv)` and
  **MUST** end it with an explicit `return 0;` (or `EXIT_SUCCESS`). C89 has no implicit
  return.
- **MUST NOT** call `gets` (removed in C11). Use `fgets`.
- Every non-`void` function **MUST** return a value on every path; `void` functions
  **MUST NOT** `return expr;`.

---

## 3. Names: avoid every keyword and reserved identifier, past and future

**MUST NOT** use any of these as identifiers (variables, functions, types, macros, members):

- C99+: `inline restrict _Bool _Complex _Imaginary`
- C11+: `_Alignas _Alignof _Atomic _Generic _Noreturn _Static_assert _Thread_local`
- C23: `bool true false nullptr static_assert alignas alignof thread_local typeof
  typeof_unqual constexpr _BitInt _Decimal32 _Decimal64 _Decimal128`
- C23 changed meaning: `auto` (do not use `auto` at all)
- C++ keywords (always avoid, cheap insurance): `class new delete this template typename
  namespace using operator private protected public virtual friend explicit mutable
  try catch throw export concept requires asm wchar_t char8_t char16_t char32_t
  noexcept decltype constinit consteval co_await co_return co_yield typeid
  static_cast dynamic_cast const_cast reinterpret_cast`
- C++ alternative tokens, which are also macros in C's `<iso646.h>` (C95): `and or not xor
  bitand bitor compl and_eq or_eq xor_eq not_eq`
- `asm` and `fortran` are listed in C89's informative "common extensions" as keywords.

**MUST NOT** define identifiers that are reserved by the standard:

- Anything beginning with `_` followed by an uppercase letter or another `_`.
- File-scope names beginning with `_`.
- Names beginning with `str`, `mem`, `wcs` + lowercase letter; `is`/`to` + lowercase letter;
  `E` + digit or uppercase letter (errno macros); `SIG`/`SIG_` + uppercase; `LC_` + uppercase.
- Any `<math.h>` function name with an `f` or `l` suffix (`sinf`, `sqrtl`): reserved since C89.
- Names reserved by later headers, even if you don't include them: `int`/`uint` + anything
  + `_t`; `INT`/`UINT` + anything + `_MAX`/`_MIN`/`_C`; `PRI`/`SCN` + lowercase letter or `X`;
  `atomic_`, `memory_order_`, `cnd_`, `mtx_`, `thrd_`, `tss_`, `FE_`, `FP_`, `TIME_` prefixes.
- Any name already in the C standard library (`abs`, `time`, `remove`, `free`, …), and the
  common POSIX names `index`, `rindex`, `bzero`, `strdup` (not ISO C, but they clash in practice).
- Not reserved in C, but reserved in C++ and best avoided: any identifier containing `__`.

**MUST** prefix all public names in a project with a short project prefix (`p_` in this
document; the user may choose another). Keep every identifier unique within its first 31
characters (the C99 minimum, met by every current toolchain). C89 strictly guarantees only
**6 significant, case-insensitive characters for external names**; if the user targets a
pre-C99 linker, make external names unique in their first 6 characters ignoring case.

---

## 4. The portability header (copy this into every project)

All compiler- and version-specific `#if` logic lives in this one file. Other files MUST NOT
test `__STDC_VERSION__`, `_MSC_VER`, `__GNUC__`, etc. directly.

```c
/* file: port.h  (universal C portability layer) */
#ifndef P_PORT_H
#define P_PORT_H

#include <limits.h>
#include <stddef.h>

/* ---- language version ------------------------------------------------ */
#if defined(__cplusplus)
#  define P_STDC 0L
#elif defined(__STDC_VERSION__)
#  define P_STDC __STDC_VERSION__
#else
#  define P_STDC 198900L
#endif

/* ---- hard platform assumptions --------------------------------------- */
#if CHAR_BIT != 8
#  error "universal C requires 8-bit bytes"
#endif

/* ---- inline / restrict ----------------------------------------------- */
#if defined(__cplusplus) || P_STDC >= 199901L
#  define P_INLINE inline
#elif defined(__GNUC__)
#  define P_INLINE __inline__
#elif defined(_MSC_VER)
#  define P_INLINE __inline
#else
#  define P_INLINE
#endif

#if !defined(__cplusplus) && P_STDC >= 199901L
#  define P_RESTRICT restrict
#elif defined(__GNUC__)
#  define P_RESTRICT __restrict__
#elif defined(_MSC_VER)
#  define P_RESTRICT __restrict
#else
#  define P_RESTRICT
#endif

/* ---- boolean (never collides with C23/C++ `bool`) --------------------- */
typedef int p_bool;
#define P_TRUE  1
#define P_FALSE 0

/* ---- exact-width integers without <stdint.h> -------------------------- */
typedef unsigned char p_u8;
typedef signed char   p_s8;

#if USHRT_MAX == 0xFFFFU
typedef unsigned short p_u16;
typedef short          p_s16;
#elif UINT_MAX == 0xFFFFU
typedef unsigned int   p_u16;
typedef int            p_s16;
#else
#  error "no 16-bit integer type"
#endif

#if UINT_MAX == 0xFFFFFFFFUL
typedef unsigned int   p_u32;
typedef int            p_s32;
#elif ULONG_MAX == 0xFFFFFFFFUL
typedef unsigned long  p_u32;
typedef long           p_s32;
#else
#  error "no 32-bit integer type"
#endif

/* 64-bit: optional. Code MUST check P_HAS_64 before using p_u64/p_s64. */
#if (!defined(__cplusplus) && P_STDC >= 199901L) || \
    (defined(__cplusplus) && __cplusplus >= 201103L)
#  include <stdint.h>
typedef uint64_t p_u64;
typedef int64_t  p_s64;
#  define P_U64_C(c) UINT64_C(c)
#  define P_HAS_64 1
#elif ((ULONG_MAX >> 31) >> 31) == 3
typedef unsigned long p_u64;
typedef long          p_s64;
#  define P_U64_C(c) c##UL
#  define P_HAS_64 1
#elif defined(_MSC_VER)
typedef unsigned __int64 p_u64;
typedef __int64          p_s64;
#  define P_U64_C(c) c##ui64
#  define P_HAS_64 1
#elif defined(__GNUC__)
/* GCC/Clang in C89 mode on ILP32 or LLP64 (32-bit Linux, MinGW): long long exists
   as an extension; __extension__ keeps -pedantic-errors quiet. */
__extension__ typedef unsigned long long p_u64;
__extension__ typedef long long          p_s64;
#  define P_U64_C(c) (__extension__ c##ULL)
#  define P_HAS_64 1
#else
#  define P_HAS_64 0
#endif

#define P_SIZE_MAX ((size_t)-1)

/* ---- utilities --------------------------------------------------------- */
#define P_CAT_(a, b) a##b
#define P_CAT(a, b)  P_CAT_(a, b)
/* File scope only. `tag` must be a unique identifier fragment. */
#define P_STATIC_ASSERT(cond, tag) \
    typedef char P_CAT(p_static_assert_, tag)[(cond) ? 1 : -1]
#define P_UNUSED(x)    ((void)(x))
#define P_ARRAY_LEN(a) (sizeof(a) / sizeof((a)[0]))

/* Wrap public headers: P_BEGIN_DECLS ... P_END_DECLS */
#ifdef __cplusplus
#  define P_BEGIN_DECLS extern "C" {
#  define P_END_DECLS   }
#else
#  define P_BEGIN_DECLS
#  define P_END_DECLS
#endif

P_STATIC_ASSERT(sizeof(p_u16) == 2, u16_size);
P_STATIC_ASSERT(sizeof(p_u32) == 4, u32_size);
P_STATIC_ASSERT((-1 & 3) == 3, twos_complement);

#endif /* P_PORT_H */
```

Notes:
- Use `P_INLINE` **only** as `static P_INLINE`. Plain `inline` means different things in
  GNU89 and C99: a non-`static` inline function in a header causes "multiple definition"
  link errors under C89 and "undefined reference" errors under C99.
- Each `P_STATIC_ASSERT` tag MUST be unique across everything a translation unit includes.
  Repeating a typedef is an error in C89/C99 (allowed only from C11).
- In C89 there may be no 64-bit type at all. Code that needs 64-bit arithmetic MUST either
  `#if P_HAS_64` with a fallback, or `#error` clearly.
- Print `p_u32` as `printf("%lu", (unsigned long)x)`. Print `p_u64` with the
  `p_u64_to_dec` helper in §5 (C89 `printf` has no 64-bit specifier).
- Print `size_t` as `printf("%lu", (unsigned long)n)`, but only when `n` is known to be
  ≤ `ULONG_MAX`. On 64-bit Windows `unsigned long` is 32-bit and the cast truncates.
  Otherwise convert to `p_u64` and use `p_u64_to_dec`. **MUST NOT** use `%zu`, `%lld`,
  `%llu`, `%hhd`, `%jd`, `%td`, `%a`, `%lf`/`%lF` in `printf` (undefined in C89; `%f`
  already takes a `double`), or `<inttypes.h>` `PRI*` macros.

---

## 5. Standard library: C89 functions only

The rule is an **allow-list**: use only the 15 headers of C89 (`<assert.h> <ctype.h>
<errno.h> <float.h> <limits.h> <locale.h> <math.h> <setjmp.h> <signal.h> <stdarg.h>
<stddef.h> <stdio.h> <stdlib.h> <string.h> <time.h>`) and only the functions and macros C89
declares in them. Newer libraries declare later additions in those same headers, and
`-std=c89` does not always hide them, so the lists below are examples, not the full set.

**MUST NOT** use these C95/C99/C11/C23 functions, macros and headers, among others:
`snprintf vsnprintf vscanf vfscanf vsscanf va_copy strtoll strtoull strtof strtold
strtoimax llabs lldiv atoll isblank _Exit`;
C99 math (`round trunc fmin fmax fma hypot log2 exp2 cbrt expm1 log1p lround nan copysign
nearbyint remainder scalbn erf tgamma lgamma`, float/long-double variants like `sinf sqrtl`,
`isnan isinf isfinite fpclassify signbit isgreater NAN INFINITY HUGE_VALF`);
C11 `aligned_alloc quick_exit at_quick_exit timespec_get` and the optional Annex K `*_s`
functions (`fopen_s strcpy_s memcpy_s sprintf_s`, which MSVC suggests);
C23 `strdup strndup memccpy memset_explicit memalignment free_sized ckd_add`;
headers `<iso646.h> <wchar.h> <wctype.h>` (C95; `-std=c89` does **not** reject these)
`<stdint.h> <inttypes.h> <stdbool.h> <complex.h> <fenv.h> <tgmath.h>
<stdalign.h> <stdatomic.h> <stdnoreturn.h> <threads.h> <uchar.h> <stdbit.h> <stdckdint.h>`
(the only exception is the guarded `<stdint.h>` inside `port.h`).

**MUST NOT** use POSIX or OS headers (`unistd.h`, `pthread.h`, `windows.h`, `sys/*.h`)
outside platform-specific files (§11).

Approved universal replacements (header, then implementation):

```c
/* file: p_util.h  (universal C helpers: declarations) */
#ifndef P_UTIL_H
#define P_UTIL_H

#include "port.h"

P_BEGIN_DECLS

p_bool p_add_int(int a, int b, int *out);
p_bool p_mul_int(int a, int b, int *out);
p_bool p_mul_size(size_t a, size_t b, size_t *out);
size_t p_strlcpy(char *dst, const char *src, size_t size);
void   p_store_u32le(unsigned char *p, p_u32 v);
p_u32  p_load_u32le(const unsigned char *p);
void   p_store_u32be(unsigned char *p, p_u32 v);
p_u32  p_load_u32be(const unsigned char *p);
p_u32  p_rotl32(p_u32 x, unsigned n);
#if P_HAS_64
char  *p_u64_to_dec(char *buf, p_u64 v);
#endif

/* Safe worst-case buffer size for sprintf("%d") of an int. Use sprintf ONLY
   when the buffer size is provably sufficient. Copy strings with p_strlcpy. */
#define P_INT_DEC_BUF (sizeof(int) * CHAR_BIT / 3 + 3)

P_END_DECLS

#endif /* P_UTIL_H */
```

```c
/* file: p_util.c  (universal C helpers: implementation) */
#include <stdlib.h>
#include <string.h>
#include "p_util.h"

/* Overflow-checked int addition. Returns P_FALSE (and leaves *out alone) on overflow. */
p_bool p_add_int(int a, int b, int *out)
{
    if ((b > 0 && a > INT_MAX - b) || (b < 0 && a < INT_MIN - b))
        return P_FALSE;
    *out = a + b;
    return P_TRUE;
}

/* Overflow-checked int multiplication. Quotients with a negative result use div(),
   which truncates toward zero in every standard; plain `/` may round toward -inf
   in C89, which would let some overflowing products through. */
p_bool p_mul_int(int a, int b, int *out)
{
    if (a > 0) {
        if (b > 0) { if (a > INT_MAX / b) return P_FALSE; }
        else       { if (b < div(INT_MIN, a).quot) return P_FALSE; }
    } else if (a < 0) {
        if (b > 0) { if (a < div(INT_MIN, b).quot) return P_FALSE; }
        else       { if (b != 0 && a < div(INT_MAX, b).quot) return P_FALSE; }
    }
    *out = a * b;
    return P_TRUE;
}

/* Overflow-checked size_t multiplication, e.g. before malloc(n * size). */
p_bool p_mul_size(size_t a, size_t b, size_t *out)
{
    if (a != 0 && b > P_SIZE_MAX / a)
        return P_FALSE;
    *out = a * b;
    return P_TRUE;
}

/* Bounded string copy (C89 has no strlcpy/snprintf). Always NUL-terminates when
   size > 0. Returns strlen(src); truncation happened if result >= size. */
size_t p_strlcpy(char *dst, const char *src, size_t size)
{
    size_t len = strlen(src);
    if (size > 0) {
        size_t n = (len < size - 1) ? len : size - 1;
        memcpy(dst, src, n);
        dst[n] = '\0';
    }
    return len;
}

/* Endian-independent serialization. Cast each byte to p_u32 BEFORE shifting:
   (p[3] << 24) would shift an int into the sign bit, which is undefined. */
void p_store_u32le(unsigned char *p, p_u32 v)
{
    p[0] = (unsigned char)(v & 0xFFu);
    p[1] = (unsigned char)((v >> 8) & 0xFFu);
    p[2] = (unsigned char)((v >> 16) & 0xFFu);
    p[3] = (unsigned char)((v >> 24) & 0xFFu);
}

p_u32 p_load_u32le(const unsigned char *p)
{
    return (p_u32)p[0]
         | ((p_u32)p[1] << 8)
         | ((p_u32)p[2] << 16)
         | ((p_u32)p[3] << 24);
}

void p_store_u32be(unsigned char *p, p_u32 v)
{
    p[0] = (unsigned char)((v >> 24) & 0xFFu);
    p[1] = (unsigned char)((v >> 16) & 0xFFu);
    p[2] = (unsigned char)((v >> 8) & 0xFFu);
    p[3] = (unsigned char)(v & 0xFFu);
}

p_u32 p_load_u32be(const unsigned char *p)
{
    return ((p_u32)p[0] << 24)
         | ((p_u32)p[1] << 16)
         | ((p_u32)p[2] << 8)
         | (p_u32)p[3];
}

/* Rotate left; n is masked so the shift count is always < 32. */
p_u32 p_rotl32(p_u32 x, unsigned n)
{
    n &= 31u;
    return (p_u32)(((x << n) | (x >> ((32u - n) & 31u))) & 0xFFFFFFFFUL);
}

#if P_HAS_64
/* Decimal formatting for 64-bit values (C89 printf cannot print them).
   buf must hold at least 21 bytes. Returns buf. */
char *p_u64_to_dec(char *buf, p_u64 v)
{
    char tmp[21];
    int i = 0, j = 0;
    do {
        tmp[i++] = (char)('0' + (int)(v % 10u));
        v /= 10u;
    } while (v != 0);
    while (i > 0)
        buf[j++] = tmp[--i];
    buf[j] = '\0';
    return buf;
}
#endif
```

Other library rules:
- `<ctype.h>`: **MUST** cast to `unsigned char`: `isspace((unsigned char)c)`.
  Passing a negative `char` is undefined.
- `getc`/`fgetc`/`getchar` result **MUST** be stored in an `int`, compared with `EOF`.
- Binary files **MUST** be opened with `"rb"`/`"wb"`/`"ab"`. Text mode translates line
  endings on Windows.
- **MUST NOT** `fwrite`/`fread` whole structs or multi-byte integers for files or
  networks. Serialize byte by byte (above).
- `sprintf` only with buffers whose worst-case size is computed (see `P_INT_DEC_BUF`).
  Never `sprintf("%s")` with input of unknown length.
- `realloc`: assign to a temporary first; never `p = realloc(p, n)`.
- **MUST NOT** depend on `malloc(0)`, `realloc(p, 0)`, `%p` output format, `rand()`
  sequences, `time_t` width, or `clock()` resolution. For reproducible randomness, write a
  PRNG on `p_u32` (e.g. xorshift32).
- `strtok`, `localtime`, `gmtime`, `asctime`, `strerror` are not reentrant. Do not use them in
  code that may run concurrently.
- `strtol`/`strtoul`: set `errno = 0` first, check `errno == ERANGE` and the end pointer.
- `qsort` is not stable: break ties explicitly in the comparator.
- Comparators **MUST NOT** return `a - b` (overflow). Use `(a > b) - (a < b)`.

---

## 6. Undefined behavior: never

Code **MUST NOT** contain or allow any of the following. Optimizers assume UB never happens and
will silently delete checks that rely on it.

**Integers**
- Signed overflow (`+ - * ++ --`, also `abs(INT_MIN)`, `-INT_MIN`, `INT_MIN / -1`,
  `INT_MIN % -1`). Check **before** the operation (`p_add_int`, `p_mul_int`). Never test
  overflow after the fact (`if (a + b < a)` is deleted by the compiler for signed types).
- Shifting by a negative amount or by ≥ the operand's width.
- Left-shifting a negative value, or shifting a 1 into/past the sign bit of a signed type.
  Do all bit manipulation on **unsigned** types.
- **Integer promotion traps.** `unsigned char`, `unsigned short`, `p_u8` and `p_u16`
  promote to **signed** `int` before arithmetic. `p_u16 a = 65535; a * a` overflows `int`
  (UB), and `(p_u8)200 << 24` shifts into the sign bit (UB). Cast to `p_u32` (or `unsigned`
  for 16-bit results) **before** multiplying or shifting: `(p_u32)a * a`.
- Division or modulo by zero.
- Converting a floating value to an integer type that cannot represent it (including
  negative → unsigned).

**Memory**
- Out-of-bounds read/write, including one-past-the-end dereference.
- Reading uninitialized automatic variables or `malloc` memory. Initialize everything.
- Use after `free`, double `free`, returning/using pointers to dead locals.
- `NULL` dereference; `memcpy`/`memset`/`strlen`/etc. with a `NULL` pointer (even with
  length 0).
- Overlapping `memcpy`/`strcpy`/`sprintf` source and destination. Use `memmove`.
- Modifying a string literal. Declare as `const char *`.
- Pointer arithmetic outside an array object (other than one-past-the-end).
- Comparing (`<`, `>`) or subtracting pointers into different objects.

**Types and aliasing**
- Accessing an object through an lvalue of an incompatible type
  (`*(p_u32 *)&some_float`, `*(p_u32 *)char_buffer`). Use `memcpy` to reinterpret bytes;
  compilers optimize it to a single load.
- Misaligned access via cast pointers. `memcpy` into a properly typed local instead.
- Calling a function through a pointer of an incompatible function type.

**Sequencing and misc**
- Modifying a variable twice, or modifying and reading it, without a sequence point:
  `i = i++`, `a[i] = i++`, `f(i++, i)`.
- `printf`/`scanf` format specifiers that don't match argument types. Cast arguments to the
  exact type the specifier expects.
- Reaching the end of a non-`void` function whose value is used.
- Data races: any unsynchronized access to shared mutable state from multiple threads.

---

## 7. Implementation-defined behavior: never depend on it

Code **MUST** produce the same result regardless of all of these:

| Never assume | Universal approach |
|---|---|
| Size of `short`/`int`/`long`/pointers | `p_u16`/`p_u32`/`p_u64`; `int` may be only 16 bits: use `long` or `p_s32` for values beyond ±32767 |
| `long` is 64-bit | It is 32-bit on Windows. Use `p_u64` |
| Pointer fits in `int`/`long` | Never store pointers in integers |
| Plain `char` is signed (or unsigned) | Use `unsigned char` for bytes, `signed char` for small signed numbers, plain `char` only for text |
| Byte order | Serialize with shifts (§5); never inspect integer bytes through pointers |
| Struct layout / padding / member offsets | Never write structs to I/O; never compare structs with `memcmp` |
| Bit-field order, signedness, and layout | Use explicit masks and shifts on unsigned integers |
| Right shift of negative values | Shift unsigned values only, or divide |
| Converting an out-of-range value to a signed type | Range-check first, or work in unsigned |
| `enum` size or underlying type | Store enums in `int` when serializing; never `sizeof(enum)` |
| Negative `/` and `%` rounding (implementation-defined in C89) | Use non-negative operands, or `div()`/`ldiv()` |
| `wchar_t` width (16-bit on Windows) | Use UTF-8 in `char` arrays |
| Large decimal constant types | Suffix constants that exceed 32767 with `L`/`UL`; write `INT_MIN`/`LONG_MIN`, never `-2147483648` |
| Execution character set is ASCII | Compare against character literals, not numbers. `'0'`..`'9'` are guaranteed contiguous; letters are not |
| Evaluation order of `f() + g()` or function arguments | Sequence side effects into separate statements |

Write integer constants with a suffix whenever they exceed 16-bit `int` range
(`70000L`, `0xFFFFFFFFUL`). Unsuffixed constants change type across standards and data
models.

---

## 8. Floating point

- **MUST NOT** compare floating values with `==`/`!=` except against exact constants you
  stored. Compare against a tolerance.
- **MUST NOT** expect bit-identical results across platforms or compilers (x87 excess
  precision, FMA contraction, `-ffast-math`, libm differences).
- When results must match bit-for-bit everywhere (lockstep simulation, money, hashing,
  consensus), use integer or fixed-point arithmetic.
- Check for NaN with `x != x` (only valid without `-ffast-math`). There is no `isnan` in C89.
- Never convert out-of-range floats to integers (UB). Clamp first.
- `printf` float output is **not** portable text. The Microsoft C runtime prints
  `%e`/`%g` with three exponent digits (`1.234568e+004` against `e+04` elsewhere), and NaN or
  infinity as `-1.#IND`/`1.#INF` against `nan`/`-nan`/`inf`. With MinGW the result even
  changes between `-std=c89` and `-std=c99`. When output must match, never `printf` a NaN or
  infinity; print with `%f` or `%.Nf`, or format fixed-point integers yourself.
- GCC contracts `a*b+c` into an FMA by default in `gnu*` modes but not in ISO `-std=c*`
  modes, so the same source can round differently depending on `-std`.

---

## 9. Translation limits (C89 minimums)

Stay within these so tiny or legacy compilers accept the code:

- Logical source line: ≤ 509 characters.
- String literal: ≤ 509 characters **after** adjacent literals are concatenated. Splitting
  `"abc" "def"` does not help (`-std=c89 -pedantic-errors` rejects it). Store longer text
  as an array of shorter strings, or as a `char` array initialized element by element.
- ≤ 31 function parameters / call arguments; ≤ 31 macro parameters / macro arguments.
- ≤ 127 members per struct/union; ≤ 127 enumerators per enum; ≤ 15 levels of nested
  struct/union definitions.
- ≤ 15 nested blocks; ≤ 8 nested `#include`s; ≤ 8 nested `#if`s.
- ≤ 32 nested parenthesized expressions; ≤ 31 nested parenthesized declarators;
  ≤ 12 pointer/array/function modifiers on one declarator.
- ≤ 257 `case` labels per `switch`.
- ≤ 511 external identifiers per translation unit; ≤ 127 identifiers declared in one
  block; ≤ 1024 macros defined at once.
- Single object ≤ 32767 bytes for guaranteed portability (larger objects: `malloc`).
- Identifiers unique within 31 characters; external names within 6 characters,
  case-insensitive, if a pre-C99 linker is a target (see §3).

---

## 10. C++ compatibility (only when the user requests it)

Additionally:
- Cast `void *` explicitly: `buf = (char *)malloc(n);`.
- Cast `int` → `enum` explicitly; never `++` an enum variable.
- No `goto` or `switch` jump past an initialized declaration.
- Do not rely on `sizeof('a')` (int in C, 1 in C++).
- Character arrays must have room for the NUL: `char s[3] = "abc";` is illegal in C++.
- No tentative definitions (`int x; int x;` at file scope).
- No `register` (removed in C++17).
- Wrap every public header's declarations in `P_BEGIN_DECLS` / `P_END_DECLS`.
- Avoid struct/enum tag names that clash with typedef or variable names.

---

## 11. When universality is impossible

Some tasks require features universal C lacks (threads, atomics, sockets, filesystems,
SIMD, high-resolution timers). In that case:

1. Tell the user which part cannot be universal and why.
2. Define a small, universal interface in a header (e.g. `p_mutex_lock(p_mutex *)`).
3. Implement it in separate per-platform files (`p_thread_posix.c`, `p_thread_win32.c`),
   selected by the build system, not by `#ifdef` scattered through the code.
4. Keep everything else universal.

Never "fix" UB with compiler flags (`-fwrapv`, `-fno-strict-aliasing`). Those only exist
on some compilers. Fix the code.

---

## 12. Checklist before outputting code

- [ ] Only `/* */` comments; all declarations at block tops; no `for (int ...)`.
- [ ] No `??` trigraph sequences or digraphs; no string literal over 509 characters after
      concatenation.
- [ ] `P_INLINE` only ever appears as `static P_INLINE`.
- [ ] `p_u8`/`p_u16`/`unsigned char`/`unsigned short` cast to `p_u32` before `*` or `<<`.
- [ ] No C99+ syntax: designated initializers, compound literals, VLAs, flexible arrays,
      variadic macros, `long long`, `bool`, bare `inline`/`restrict`, trailing enum comma.
- [ ] Every function has a return type and a prior prototype; empty lists are `(void)`.
- [ ] `main` returns explicitly.
- [ ] No identifier is a C99/C11/C23/C++ keyword or reserved name; public names prefixed.
- [ ] Only C89 library functions and headers (plus `port.h`).
- [ ] `printf`/`scanf` specifiers are C89 and arguments are cast to match exactly.
- [ ] Every signed arithmetic operation that could overflow is checked beforehand.
- [ ] All bit ops and shifts are on unsigned types with in-range shift counts.
- [ ] Bytes are `unsigned char`; `<ctype.h>` arguments cast to `unsigned char`;
      `getc` results stored in `int`.
- [ ] Every array index and pointer is bounds-checked; every variable initialized.
- [ ] No type punning through pointers; `memcpy` used for reinterpretation.
- [ ] No struct or integer written raw to I/O; serialization is explicit-endian.
- [ ] No assumption about `int`/`long` width, `char` signedness, endianness, padding,
      or evaluation order.
- [ ] No floating `==`; no float → int conversion without range check.
- [ ] Non-portable code isolated behind an interface in per-platform files.
- [ ] Every `malloc`/`realloc`/`fopen` result checked; size computations overflow-checked.

---

## 13. Verification the user should run (suggest this when delivering code)

```sh
F="-Wall -Wextra -pedantic-errors -Werror -Wconversion -Wsign-conversion -Wshadow \
   -Wformat=2 -Wcast-align -Wstrict-prototypes -Wmissing-prototypes"

# 1. Compiles under every standard
for s in c89 c99 c11 c17 c2x; do
  gcc   -std=$s $F -c *.c || exit 1
  clang -std=$s $F -c *.c || exit 1
done

# 2. Behaves correctly with sanitizers (run the test suite on each)
gcc -std=c89 -O1 -g -fsanitize=undefined,address -fno-sanitize-recover=all *.c -o t && ./t
clang -std=c89 -O1 -g -fsanitize=memory *.c -o t && ./t

# 3. Platform variation
gcc -std=c89 -O2 -funsigned-char *.c -o t && ./t     # unsigned plain char
gcc -std=c89 -O2 -m32 *.c -o t && ./t                 # 32-bit
x86_64-w64-mingw32-gcc -std=c89 -O2 *.c -o t.exe && wine t.exe   # LLP64
s390x-linux-gnu-gcc -std=c89 -O2 -static *.c -o t && qemu-s390x ./t  # big-endian

# 4. Static analysis
gcc -fanalyzer -c *.c ; cppcheck --std=c89 --enable=all .

# 5. Things -std=c89 does not reject: C95 headers and trigraphs
grep -nE '#[[:space:]]*include[[:space:]]*<(iso646|wchar|wctype)\.h>|\?\?[=(/)'"'"'<!>-]' *.c *.h \
  && echo "non-universal construct found" && exit 1
```

Same inputs MUST produce identical outputs on every configuration. For text streams that
means identical after line-ending normalization: Windows text mode writes `\n` as `\r\n`.
Where bytes must match exactly, open the file with `"wb"` and write `\n` yourself. For high-assurance
code, also consult SEI CERT C and MISRA C, and consider Frama-C for proofs of UB absence.

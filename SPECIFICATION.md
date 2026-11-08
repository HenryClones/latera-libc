# Freestanding C Standard Library Specification

This document defines the formal interface contracts, standard taxonomy, type requirements, function signatures, edge-case semantics, and runtime execution invariants for the freestanding C standard library (`libc`).

---

## 1. Standards Scope & Header Taxonomy

The C standard distinguishes between **hosted** and **freestanding** environments:
- In **ISO C11** (§4.6), a freestanding environment relies on the compiler toolchain to provide fundamental freestanding headers (`<float.h>`, `<iso646.h>`, `<limits.h>`, `<stdalign.h>`, `<stdarg.h>`, `<stdbool.h>`, `<stddef.h>`, `<stdint.h>`, and `<stdnoreturn.h>`).
- In **ISO C23** (§4.6), freestanding requirements are expanded to include `<stdckdint.h>`, `<stdbit.h>`, a constrained subset of `<string.h>`, and an arithmetic/sorting subset of `<stdlib.h>`.

To support bare-metal kernels, embedded micro-runtimes, and user-space concurrency engines without pulling in host dependencies, this specification defines both mandated freestanding headers and select project-supported hosted facilities:

| Header | Standard Category | Specification Scope |
| :--- | :--- | :--- |
| `<stddef.h>` | ISO Mandated (Freestanding C11) | C11-complete types, macros, and constants (C23 additions like `nullptr_t`/`unreachable()` deferred). |
| `<stdint.h>` | ISO Mandated (Freestanding C11) | C11-complete exact, least, fast, pointer, and greatest-width types, limits, and constant macros (C23 `*_WIDTH` macros deferred). |
| `<string.h>` | Mandated in C23 Freestanding / Hosted in C11 | Memory block operations and string inspection routines. |
| `<stdlib.h>` | Hosted Facility (Select Subset Provided) | Integer arithmetic, numeric parsing, sorting/searching, pluggable dynamic memory contracts, and termination hooks. |
| `<assert.h>` | Hosted Facility (Diagnostic Facility Provided) | Diagnostic assertion macro with dual C11/C23 semantics, multiple inclusion support, and pluggable failure hooks. |
| `<memory.h>` | Non-Standard Extension | Compatibility header aliasing `<string.h>` memory operations (`memcpy`, `memmove`, `memset`, `memcmp`, `memchr`). |

### 1.1 Compiler-Provided Headers & Mandatory Libcall Primitives
- **Compiler-Provided Freestanding Headers:** In accordance with ISO C freestanding rules, headers defining compiler type representations and machine limits—specifically `<limits.h>`, `<float.h>`, `<stdarg.h>`, `<stdbool.h>`, `<stdalign.h>`, and `<stdnoreturn.h>`—are supplied directly by the compiler toolchain (`gcc`/`clang`). This libc relies on compiler-provided `<limits.h>` for fundamental type limits (`INT_MIN`, `LONG_MAX`, etc.).
- **Mandatory Compiler Libcall Primitives:** Under `-ffreestanding`, optimizing compilers (such as GCC and Clang) may synthesize calls to `memcpy`, `memmove`, `memset`, and `memcmp` for struct assignments, bulk initializations, and block comparisons. Therefore, a conforming freestanding libc must unconditionally provide these four fundamental primitives.

### 1.2 Core Execution Invariants
- **Zero Host Syscalls:** Standard library routines never issue direct host operating system syscalls. Any platform interaction (heap expansion, diagnostic reporting, process termination) delegates to pluggable environment hooks or supervisor traps.
- **Static Resolution Only:** All routines resolve strictly through static linkage (`-static`); dynamic symbol resolution, GOT/PLT indirections, and dynamic shared objects are strictly disallowed.
- **Zero Host TLS Dependency:** Core routines do not depend on compiler thread-local storage (`_Thread_local` / `__thread`) or host stack-canary TLS layouts (`%fs:0x28`).
- **No Unresolved External Symbols:** Clean self-contained implementation with zero implicit dependencies on host libc startup files (`crt0.o`) or runtime symbols.

---

## 2. Standard Header Specifications

### 2.1 `<stddef.h>`

Defines fundamental primitive types, constants, and structure offset macros for freestanding environments.

#### Types (x86-64 LP64)
- `size_t`: Unsigned integer type of the result of the `sizeof` operator (`unsigned long` on x86-64).
- `ptrdiff_t`: Signed integer type of the result of subtracting two pointers (`long` on x86-64).
- `max_align_t`: Type whose alignment requirement is at least as strict as that of every scalar type (`long double` or 16-byte aligned struct on x86-64).
- `wchar_t`: Integer type whose range of values can represent distinct wide character codes (`int` on x86-64).

#### Constants & Macros
- `NULL`: Null pointer constant defined as `((void *)0)` or `0`.
- `offsetof(type, member)`: Integral constant expression of type `size_t`, evaluating to the offset in bytes from the structure start to `member`.

*Note: C23 additions (`nullptr_t`, `unreachable()`, and `__STDC_VERSION_STDDEF_H__`) are deferred to future revisions.*

---

### 2.2 `<stdint.h>`

Defines exact-width, minimum-width, fastest minimum-width, pointer-holding, and greatest-width integer types, along with corresponding limit and constant macros.

#### Exact-Width Integer Types (x86-64 LP64)
- Signed:
  - `int8_t`: `signed char`
  - `int16_t`: `short`
  - `int32_t`: `int`
  - `int64_t`: `long`
- Unsigned:
  - `uint8_t`: `unsigned char`
  - `uint16_t`: `unsigned short`
  - `uint32_t`: `unsigned int`
  - `uint64_t`: `unsigned long`

#### Minimum-Width Integer Types (`int_leastN_t`, `uint_leastN_t`)
- Signed: `int_least8_t` (`signed char`), `int_least16_t` (`short`), `int_least32_t` (`int`), `int_least64_t` (`long`)
- Unsigned: `uint_least8_t` (`unsigned char`), `uint_least16_t` (`unsigned short`), `uint_least32_t` (`unsigned int`), `uint_least64_t` (`unsigned long`)

#### Fastest Minimum-Width Integer Types (`int_fastN_t`, `uint_fastN_t`)
- Signed: `int_fast8_t` (`signed char`), `int_fast16_t` (`long`), `int_fast32_t` (`long`), `int_fast64_t` (`long`)
- Unsigned: `uint_fast8_t` (`unsigned char`), `uint_fast16_t` (`unsigned long`), `uint_fast32_t` (`unsigned long`), `uint_fast64_t` (`unsigned long`)

#### Pointer-Holding & Greatest-Width Types
- `intptr_t`, `uintptr_t`: `long`, `unsigned long`
- `intmax_t`, `uintmax_t`: `long`, `unsigned long`

#### Limit Macros
Limit macros evaluate to constant expressions of the required promoted type (ISO C11 §7.20.2p1):
- Exact-width limits:
  - `INT8_MIN`: `(-128)` (type `int`)
  - `INT8_MAX`: `127` (type `int`)
  - `UINT8_MAX`: `255` (type `int`, matching promotion of `uint8_t`)
  - `INT16_MIN`: `(-32768)` (type `int`)
  - `INT16_MAX`: `32767` (type `int`)
  - `UINT16_MAX`: `65535` (type `int`, matching promotion of `uint16_t`)
  - `INT32_MIN`: `(-2147483647 - 1)` (type `int`)
  - `INT32_MAX`: `2147483647` (type `int`)
  - `UINT32_MAX`: `4294967295U` (type `unsigned int`)
  - `INT64_MIN`: `(-9223372036854775807L - 1)` (type `long`)
  - `INT64_MAX`: `9223372036854775807L` (type `long`)
  - `UINT64_MAX`: `18446744073709551615UL` (type `unsigned long`)
- Least-width limits: `INT_LEAST8_MIN`/`MAX`, `UINT_LEAST8_MAX`, `INT_LEAST16_MIN`/`MAX`, `UINT_LEAST16_MAX`, `INT_LEAST32_MIN`/`MAX`, `UINT_LEAST32_MAX`, `INT_LEAST64_MIN`/`MAX`, `UINT_LEAST64_MAX`
- Fast-width limits: `INT_FAST8_MIN`/`MAX`, `UINT_FAST8_MAX`, `INT_FAST16_MIN`/`MAX`, `UINT_FAST16_MAX`, `INT_FAST32_MIN`/`MAX`, `UINT_FAST32_MAX`, `INT_FAST64_MIN`/`MAX`, `UINT_FAST64_MAX`
- Pointer limits: `INTPTR_MIN` (`-9223372036854775807L - 1`), `INTPTR_MAX` (`9223372036854775807L`), `UINTPTR_MAX` (`18446744073709551615UL`)
- Greatest-width limits: `INTMAX_MIN` (`-9223372036854775807L - 1`), `INTMAX_MAX` (`9223372036854775807L`), `UINTMAX_MAX` (`18446744073709551615UL`)
- Other limits:
  - `PTRDIFF_MIN` (`-9223372036854775807L - 1`), `PTRDIFF_MAX` (`9223372036854775807L`)
  - `SIZE_MAX` (`18446744073709551615UL`)
  - `SIG_ATOMIC_MIN` (`(-2147483647 - 1)`), `SIG_ATOMIC_MAX` (`2147483647`)
  - `WCHAR_MIN` (`(-2147483647 - 1)`), `WCHAR_MAX` (`2147483647`)
  - `WINT_MIN` (`0U`), `WINT_MAX` (`4294967295U`)

#### Constant Macros
- `INT8_C(value)`, `INT16_C(value)`: Expand to integer constants of promoted type `int` (`value`)
- `UINT8_C(value)`, `UINT16_C(value)`: Expand to integer constants of promoted type `int` (`value`)
- `INT32_C(value)`: `value`
- `UINT32_C(value)`: `value ## U`
- `INT64_C(value)`: `value ## L`
- `UINT64_C(value)`: `value ## UL`
- `INTMAX_C(value)`: `value ## L`
- `UINTMAX_C(value)`: `value ## UL`

*Note: C23 additions (`*_WIDTH` macros such as `INT8_WIDTH`, `UINT64_WIDTH`, `SIZE_WIDTH`, etc. and `__STDC_VERSION_STDINT_H__`) are deferred.*

---

### 2.3 `<string.h>` & `<memory.h>`

Defines byte-level memory operations and null-terminated string inspection routines. `<memory.h>` is a compatibility extension providing the standard memory manipulation functions.

#### Memory Operations
```c
void *memcpy(void *dest, const void *src, size_t n);
```
- Copies `n` bytes from `src` to `dest`. Memory areas must not overlap.
- **Pointer Validity & `n == 0` Semantics:** When `n == 0`, no bytes are copied and `dest` is returned immediately. Under ISO C11/C23 (§7.1.4, §7.24.1p2), passing `NULL` for `dest` or `src` is undefined behavior regardless of `n`.

```c
void *memmove(void *dest, const void *src, size_t n);
```
- Copies `n` bytes from `src` to `dest`, safely handling overlapping memory regions.
- **Pointer Validity & `n == 0` Semantics:** When `n == 0`, no bytes are copied and `dest` is returned immediately. Passing `NULL` for `dest` or `src` is undefined behavior regardless of `n`.

```c
void *memset(void *s, int c, size_t n);
```
- Sets the first `n` bytes of the memory area `s` to the value `(unsigned char)c`.
- **Pointer Validity & `n == 0` Semantics:** When `n == 0`, no bytes are written and `s` is returned immediately. Passing `NULL` for `s` is undefined behavior regardless of `n`.

```c
int memcmp(const void *s1, const void *s2, size_t n);
```
- Compares the first `n` bytes of `s1` and `s2` as unsigned characters.
- **Return Value & Pointer Validity:** Returns `0` if all `n` bytes match (or if `n == 0`); `< 0` if the first differing byte in `s1` is less than `s2`; `> 0` if greater. Passing `NULL` for `s1` or `s2` is undefined behavior regardless of `n`.

```c
void *memchr(const void *s, int c, size_t n);
```
- Searches the first `n` bytes of `s` for the first occurrence of `(unsigned char)c`.
- **Return Value & Pointer Validity:** Pointer to matching byte, or `NULL` if not found (or if `n == 0`). Passing `NULL` for `s` is undefined behavior regardless of `n`.

#### String Operations
```c
size_t strlen(const char *s);
```
- Computes length of null-terminated string `s` excluding terminating `\0`.

```c
int strcmp(const char *s1, const char *s2);
int strncmp(const char *s1, const char *s2, size_t n);
```
- Lexicographically compares null-terminated strings `s1` and `s2` (up to `n` characters for `strncmp`).
- **Return Value & Pointer Validity:** `< 0`, `0`, or `> 0` based on `(unsigned char)` character differences. If `n == 0` in `strncmp`, returns `0`. Passing `NULL` for `s1` or `s2` is undefined behavior.

```c
char *strcpy(char *dest, const char *src);
char *strncpy(char *dest, const char *src, size_t n);
```
- `strcpy`: Copies null-terminated string `src` to `dest` (including `\0`). Returns `dest`.
- `strncpy`: Copies at most `n` characters. If `strlen(src) < n`, pads remainder of `dest` with null bytes `\0`. If `strlen(src) >= n`, `dest` is not null-terminated. Returns `dest`.
- **Pointer Validity:** Passing `NULL` for `dest` or `src` is undefined behavior.

```c
char *strcat(char *dest, const char *src);
char *strncat(char *dest, const char *src, size_t n);
```
- `strcat`: Appends `src` to `dest`, overwriting `dest`'s null terminator and adding a new null terminator. Returns `dest`.
- `strncat`: Appends at most `n` characters from `src` to `dest`, always appending a terminating null byte `\0`. Returns `dest`.
- **Pointer Validity:** Passing `NULL` for `dest` or `src` is undefined behavior.

```c
char *strchr(const char *s, int c);
char *strrchr(const char *s, int c);
```
- `strchr`: Locates first occurrence of `(char)c` in `s`.
- `strrchr`: Locates last occurrence of `(char)c` in `s`.
- **Edge cases:** When `c == '\0'`, returns pointer to the terminating null byte of string `s`. Returns `NULL` if character is not found.

```c
char *strstr(const char *haystack, const char *needle);
```
- Finds the first occurrence of substring `needle` in `haystack`.
- **Edge cases:** If `needle` is an empty string (`""`), returns `(char *)haystack`. If `needle` is not found, returns `NULL`.

---

### 2.4 `<stdlib.h>`

Defines integer conversions, arithmetic, searching/sorting, pluggable allocator contracts, and termination hooks.

#### Numeric Conversions
```c
int atoi(const char *nptr);
long atol(const char *nptr);
long long atoll(const char *nptr);
```
- Parses initial digits in `nptr` as a base-10 signed integer. Leading whitespace is skipped.
- `atoi`: Equivalent to `(int)strtol(nptr, (char **)NULL, 10)` without error reporting.
- `atol`: Equivalent to `strtol(nptr, (char **)NULL, 10)` without error reporting.
- `atoll`: Equivalent to `strtoll(nptr, (char **)NULL, 10)` without error reporting.
- **Edge cases & Errors:** If the converted value is out of range, behavior is undefined (consistent with ISO C).

```c
long strtol(const char *nptr, char **endptr, int base);
unsigned long strtoul(const char *nptr, char **endptr, int base);
long long strtoll(const char *nptr, char **endptr, int base);
unsigned long long strtoull(const char *nptr, char **endptr, int base);
```
- Parses string `nptr` according to `base` (valid bases: `0`, `2`..`36`).
- **Base 0 Behavior:** Autodetects radix (`0x`/`0X` for hexadecimal, `0` for octal, decimal otherwise).
- **Prefixes:** If base is 16 (or 0 with hex), optional `0x`/`0X` prefix is recognized and skipped.
- **Pointer Storing (`endptr`):** If `endptr != NULL`, stores pointer to character where parsing stopped. If no digits were found, `*endptr` is set to `nptr` and returns `0`.
- **Overflow Clamping:** In freestanding mode (where `<errno.h>` is not mandated), values exceeding type ranges clamp to type boundaries:
  - `strtol`: `LONG_MAX` or `LONG_MIN`
  - `strtoul`: `ULONG_MAX`
  - `strtoll`: `LLONG_MAX` or `LLONG_MIN`
  - `strtoull`: `ULLONG_MAX`

#### Integer Arithmetic
```c
int abs(int j);
long labs(long j);
long long llabs(long long j);
```
- Computes absolute value of integer. Note: `abs(INT_MIN)`, `labs(LONG_MIN)`, and `llabs(LLONG_MIN)` produce undefined behavior due to two's complement asymmetry.

```c
typedef struct { int quot; int rem; } div_t;
typedef struct { long quot; long rem; } ldiv_t;
typedef struct { long long quot; long long rem; } lldiv_t;

div_t div(int numer, int denom);
ldiv_t ldiv(long numer, long denom);
lldiv_t lldiv(long long numer, long long denom);
```
- Computes algebraic quotient and remainder such that `numer = quot * denom + rem`, where `rem` has the same sign as `numer` and `|rem| < |denom|`.
- **Edge cases:** Division by zero (`denom == 0`) or overflow on dividing minimum value by `-1` (`INT_MIN / -1` for `div`, `LONG_MIN / -1` for `ldiv`, `LLONG_MIN / -1` for `lldiv`) produces undefined behavior / hardware arithmetic trap.

#### Searching & Sorting
```c
void qsort(void *base, size_t nmemb, size_t size,
           int (*compar)(const void *, const void *));
```
- Sorts array `base` of `nmemb` elements of size `size` in place using comparison function `compar`.
- **Edge cases:** If `nmemb == 0` or `size == 0`, returns immediately without inspecting memory.

```c
void *bsearch(const void *key, const void *base, size_t nmemb, size_t size,
              int (*compar)(const void *, const void *));
```
- Searches sorted array `base` for element matching `key`. Returns pointer to matching element or `NULL`.

#### Pluggable Dynamic Memory Allocation Contracts
```c
void *malloc(size_t size);
void free(void *ptr);
void *calloc(size_t nmemb, size_t size);
void *realloc(void *ptr, size_t size);
```
- Dynamic memory management in freestanding libc is mediated by pluggable allocator backends (e.g. bump, slab, or buddy allocators) via the `__libc_heap_sbrk` hook. If no allocator hook is configured or if memory is exhausted, allocation functions deterministically return `NULL`.
- **Edge cases:**
  - `malloc(0)` returns `NULL`.
  - `free(NULL)` is a safe no-op.
  - `calloc`: Returns zero-initialized memory. If `nmemb * size` overflows `size_t`, fails and returns `NULL`.
  - `realloc(NULL, size)` is equivalent to `malloc(size)`.
  - `realloc(ptr, 0)` frees `ptr` and returns `NULL` (explicit implementation choice; note that ISO C23 treats `realloc(ptr, 0)` as undefined behavior).

#### Execution Control & Termination Hooks
```c
_Noreturn void abort(void);
_Noreturn void exit(int status);
```
- `abort()`: Causes abnormal execution termination by invoking the pluggable supervisor fault hook (`__libc_abort`). Does not return.
- `exit(status)`: Yields control to supervisor runtime with exit code `status` via `__libc_exit(status)`. Does not return.

---

### 2.5 `<assert.h>`

Defines diagnostic assertion facilities for compile-time and runtime validation.

#### Multiple Inclusion Semantics & Version Support
`<assert.h>` is explicitly designed without standard `#ifndef` include guards. Each time `<assert.h>` is included, it undefines `assert` (`#undef assert`) and redefines it according to the presence or absence of the `NDEBUG` macro at that specific point in the translation unit:

```c
#undef assert

#if defined(__STDC_VERSION__) && __STDC_VERSION__ >= 202311L
  /* ISO C23: assert is variadic and static_assert is a standard language keyword */
  #ifdef NDEBUG
    #define assert(...) ((void)0)
  #else
    #define assert(...) \
        ((__VA_ARGS__) ? (void)0 : __assert_fail(#__VA_ARGS__, __FILE__, __LINE__, __func__))
  #endif
#else
  /* ISO C11 / C17: assert takes a single expression and static_assert is a macro */
  #ifdef NDEBUG
    #define assert(expr) ((void)0)
  #else
    #define assert(expr) \
        ((expr) ? (void)0 : __assert_fail(#expr, __FILE__, __LINE__, __func__))
  #endif

  #ifndef static_assert
    #define static_assert _Static_assert
  #endif
#endif
```

#### Pluggable Diagnostic Hook Contract
```c
_Noreturn void __assert_fail(const char *assertion, const char *file,
                             unsigned int line, const char *function);
```
- **Contract:** Invoked when an assertion evaluates to false. A default weak implementation dispatches to `abort()`. Consumer runtimes (kernel, test harnesses, user runtimes) may override `__assert_fail` to output diagnostics to their respective log or supervisor interfaces without tying the libc specification to specific hardware devices.

---

## 3. Pluggable Runtime Hooks & Platform Boundary

Freestanding execution decouples software algorithms from execution environments. The following runtime hooks define the boundary between libc and consumer platforms:

```c
/* Execution Control Hooks */
_Noreturn void __libc_abort(void);
_Noreturn void __libc_exit(int status);

/* Diagnostic Hook */
_Noreturn void __assert_fail(const char *assertion, const char *file,
                             unsigned int line, const char *function);

/* Memory Backend Hook (Optional Pluggable Memory Provider) */
void *__libc_heap_sbrk(ptrdiff_t increment);
```

### Hook Status, Fallbacks & Lifecycles
- **`__libc_abort()`**: Weak default implementation executes a portable infinite halt loop (`for (;;) { }`). Consumer platforms (kernel panic handlers, test runners) may override this hook to provide custom abort behavior.
- **`__libc_exit(int status)`**: Weak default implementation delegates directly to `__libc_abort()`.
- **`__assert_fail(...)`**: Weak default implementation delegates directly to `__libc_abort()`. Consumer runtimes may override `__assert_fail` to emit diagnostic logging before halting.
- **`__libc_heap_sbrk(ptrdiff_t increment)`**: Optional weak symbol. If not supplied by consumer (symbol resolves to `NULL`) or if it returns `(void *)-1`, dynamic memory allocation functions (`malloc`, `calloc`, `realloc`) deterministically return `NULL`.
- **Startup / Entry Point:** Freestanding code compiled with `-nostdlib` contains no standard `_start` or `main` startup wrapper. Execution begins directly at the consumer platform's designated entry point (e.g. EFI entry, kernel entry, or fiber entry).

---

## 4. Linkage & ABI Requirements

1. **Target Architecture:** x86-64 (AMD64) System V ABI calling convention (integer parameters in `%rdi`, `%rsi`, `%rdx`, `%rcx`, `%r8`, `%r9`; return value in `%rax`/`%rdx`).
2. **Stack Alignment:** Callers and callees must maintain 16-byte stack alignment prior to `call` instructions.
3. **Red-Zone Constraints:**
   - In **Ring 0** (kernel/storage/interrupt context), code MUST compile with `-mno-red-zone` to prevent asynchronous interrupts from overwriting stack frames below `%rsp`.
   - In **Ring 3** (user-space payloads), standard 128-byte red zone may be utilized.
4. **Stack Protection:**
   - Ring 0: `-fno-stack-protector` mandatory.
   - Ring 3: `-fno-stack-protector` recommended unless consumer provides `__stack_chk_guard` and `__stack_chk_fail`.
5. **Static Linkage:** Strictly `-static` resolution. No undefined references to dynamic loader symbols or host runtime libraries.

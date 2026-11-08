# Freestanding C Standard Library Specification

This document defines the formal interface contracts, standard taxonomy, type requirements, function signatures, edge-case semantics, and runtime execution invariants for the freestanding C standard library (`libc`).

---

## 1. Standards Scope & Header Taxonomy

The C standard distinguishes between **hosted** and **freestanding** environments:
- In **ISO C11** (§4.6), a freestanding environment is only mandated to provide `<float.h>`, `<iso646.h>`, `<limits.h>`, `<stdalign.h>`, `<stdarg.h>`, `<stdbool.h>`, `<stddef.h>`, `<stdint.h>`, and `<stdnoreturn.h>`.
- In **ISO C23** (§4.6), freestanding requirements are expanded to include `<stdckdint.h>`, `<stdbit.h>`, a constrained subset of `<string.h>`, and an arithmetic/sorting subset of `<stdlib.h>`.

To support bare-metal kernels, embedded micro-runtimes, and user-space concurrency engines without pulling in host dependencies, this specification defines both mandated freestanding headers and select project-supported hosted facilities:

| Header | Standard Category | Specification Scope |
| :--- | :--- | :--- |
| `<stddef.h>` | ISO Mandated (C11/C23 Freestanding) | Complete standard types, macros, and constants. |
| `<stdint.h>` | ISO Mandated (C11/C23 Freestanding) | Complete exact, least, fast, pointer, and greatest-width types, limits, and constant macros. |
| `<string.h>` | Mandated in C23 Freestanding / Extended in C11 | Memory block operations and string inspection routines. |
| `<stdlib.h>` | Hosted Facility (Select Subset Provided) | Integer arithmetic, numeric parsing, sorting/searching, pluggable dynamic memory contracts, and termination hooks. |
| `<assert.h>` | Hosted Facility (Diagnostic Facility Provided) | Diagnostic assertion macro, C11/C23 `static_assert` support, multiple inclusion semantics, and pluggable failure hooks. |
| `<memory.h>` | Non-Standard Extension | Compatibility header aliasing `<string.h>` memory operations (`memcpy`, `memmove`, `memset`, `memcmp`, `memchr`). |

### 1.1 Core Execution Invariants
- **Zero Host Syscalls:** Standard library routines never issue direct host operating system syscalls. Any platform interaction (heap expansion, diagnostic reporting, process termination) delegates to pluggable environment hooks or supervisor traps.
- **Static Resolution Only:** All routines resolve strictly through static linkage (`-static`); dynamic symbol resolution, GOT/PLT indirections, and dynamic shared objects are strictly disallowed.
- **Zero Host TLS Dependency:** Core routines do not depend on compiler thread-local storage (`_Thread_local` / `__thread`) or host stack-canary TLS layouts (`%fs:0x28`).
- **No Unresolved External Symbols:** Clean self-contained implementation with zero implicit dependencies on host libc startup files (`crt0.o`) or runtime symbols.

---

## 2. Standard Header Specifications

### 2.1 `<stddef.h>`

Defines fundamental primitive types, constants, and structure offset macros.

#### Types
- `size_t`: Unsigned integer type of the result of the `sizeof` operator (`unsigned long` on x86-64).
- `ptrdiff_t`: Signed integer type of the result of subtracting two pointers (`long` on x86-64).
- `max_align_t`: Type whose alignment requirement is at least as strict as that of every scalar type (`long double` or 16-byte aligned struct on x86-64).
- `wchar_t`: Integer type whose range of values can represent distinct wide character codes (`int` on x86-64).

#### Constants & Macros
- `NULL`: Null pointer constant defined as `((void *)0)` or `0`.
- `offsetof(type, member)`: Integral constant expression of type `size_t`, evaluating to the offset in bytes from the structure start to `member`.

---

### 2.2 `<stdint.h>`

Defines exact-width, minimum-width, fastest minimum-width, pointer-holding, and greatest-width integer types, along with corresponding limit and constant macros.

#### Exact-Width Integer Types
- Signed: `int8_t`, `int16_t`, `int32_t`, `int64_t`
- Unsigned: `uint8_t`, `uint16_t`, `uint32_t`, `uint64_t`

#### Minimum-Width Integer Types (`int_leastN_t`, `uint_leastN_t`)
- Signed: `int_least8_t`, `int_least16_t`, `int_least32_t`, `int_least64_t`
- Unsigned: `uint_least8_t`, `uint_least16_t`, `uint_least32_t`, `uint_least64_t`

#### Fastest Minimum-Width Integer Types (`int_fastN_t`, `uint_fastN_t`)
- Signed: `int_fast8_t`, `int_fast16_t`, `int_fast32_t`, `int_fast64_t`
- Unsigned: `uint_fast8_t`, `uint_fast16_t`, `uint_fast32_t`, `uint_fast64_t`

#### Pointer-Holding & Greatest-Width Types
- `intptr_t`, `uintptr_t`: Signed/unsigned integer types capable of holding any `void *` pointer without truncation.
- `intmax_t`, `uintmax_t`: Signed/unsigned integer types of greatest width supported by the target.

#### Limit Macros
- Exact-width limits:
  - `INT8_MIN` (`-128`), `INT8_MAX` (`127`), `UINT8_MAX` (`255U`)
  - `INT16_MIN` (`-32768`), `INT16_MAX` (`32767`), `UINT16_MAX` (`65535U`)
  - `INT32_MIN` (`-2147483648`), `INT32_MAX` (`2147483647`), `UINT32_MAX` (`4294967295U`)
  - `INT64_MIN` (`-9223372036854775808LL`), `INT64_MAX` (`9223372036854775807LL`), `UINT64_MAX` (`18446744073709551615ULL`)
- Least-width limits: `INT_LEAST8_MIN`/`MAX`, `UINT_LEAST8_MAX`, `INT_LEAST16_MIN`/`MAX`, `UINT_LEAST16_MAX`, `INT_LEAST32_MIN`/`MAX`, `UINT_LEAST32_MAX`, `INT_LEAST64_MIN`/`MAX`, `UINT_LEAST64_MAX`
- Fast-width limits: `INT_FAST8_MIN`/`MAX`, `UINT_FAST8_MAX`, `INT_FAST16_MIN`/`MAX`, `UINT_FAST16_MAX`, `INT_FAST32_MIN`/`MAX`, `UINT_FAST32_MAX`, `INT_FAST64_MIN`/`MAX`, `UINT_FAST64_MAX`
- Pointer limits: `INTPTR_MIN`, `INTPTR_MAX`, `UINTPTR_MAX`
- Greatest-width limits: `INTMAX_MIN`, `INTMAX_MAX`, `UINTMAX_MAX`
- Other limits:
  - `PTRDIFF_MIN`, `PTRDIFF_MAX`
  - `SIZE_MAX`
  - `SIG_ATOMIC_MIN`, `SIG_ATOMIC_MAX`
  - `WCHAR_MIN`, `WCHAR_MAX`
  - `WINT_MIN`, `WINT_MAX`

#### Constant Macros
- `INT8_C(value)`, `INT16_C(value)`, `INT32_C(value)`, `INT64_C(value)`
- `UINT8_C(value)`, `UINT16_C(value)`, `UINT32_C(value)`, `UINT64_C(value)`
- `INTMAX_C(value)`, `UINTMAX_C(value)`

---

### 2.3 `<string.h>` & `<memory.h>`

Defines byte-level memory operations and null-terminated string inspection routines. `<memory.h>` is a compatibility extension providing the five standard memory manipulation functions.

#### Memory Operations
```c
void *memcpy(void *dest, const void *src, size_t n);
```
- Copies `n` bytes from `src` to `dest`. Memory areas must not overlap.
- **Edge cases:** When `n == 0`, no bytes are copied and `dest` is returned immediately. `dest` and `src` must be valid pointers when `n > 0`.

```c
void *memmove(void *dest, const void *src, size_t n);
```
- Copies `n` bytes from `src` to `dest`, safely handling overlapping memory regions.
- **Edge cases:** When `n == 0`, no bytes are copied and `dest` is returned immediately.

```c
void *memset(void *s, int c, size_t n);
```
- Sets the first `n` bytes of the memory area `s` to the value `(unsigned char)c`.
- **Edge cases:** When `n == 0`, no bytes are written and `s` is returned immediately.

```c
int memcmp(const void *s1, const void *s2, size_t n);
```
- Compares the first `n` bytes of `s1` and `s2` as unsigned characters.
- **Return Value:** Returns `0` if all `n` bytes match (or if `n == 0`); `< 0` if the first differing byte in `s1` is less than `s2`; `> 0` if greater.

```c
void *memchr(const void *s, int c, size_t n);
```
- Searches the first `n` bytes of `s` for the first occurrence of `(unsigned char)c`.
- **Return Value:** Pointer to matching byte, or `NULL` if not found (or if `n == 0`).

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
- **Return Value:** `< 0`, `0`, or `> 0` based on `(unsigned char)` character differences. If `n == 0` in `strncmp`, returns `0`.

```c
char *strcpy(char *dest, const char *src);
char *strncpy(char *dest, const char *src, size_t n);
```
- `strcpy`: Copies null-terminated string `src` to `dest` (including `\0`). Returns `dest`.
- `strncpy`: Copies at most `n` characters. If `strlen(src) < n`, pads remainder of `dest` with null bytes `\0`. If `strlen(src) >= n`, `dest` is not null-terminated. Returns `dest`.

```c
char *strcat(char *dest, const char *src);
char *strncat(char *dest, const char *src, size_t n);
```
- `strcat`: Appends `src` to `dest`, overwriting `dest`'s null terminator and adding a new null terminator. Returns `dest`.
- `strncat`: Appends at most `n` characters from `src` to `dest`, always appending a terminating null byte `\0`. Returns `dest`.

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
- **Edge cases & Errors:** If the converted value is out of range, behavior is undefined (consistent with ISO C). Equivalent to `(type)strtol(nptr, (char **)NULL, 10)` without error reporting.

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
- Computes absolute value of integer. Note: `abs(INT_MIN)` produces undefined behavior due to two's complement asymmetry.

```c
typedef struct { int quot; int rem; } div_t;
typedef struct { long quot; long rem; } ldiv_t;
typedef struct { long long quot; long long rem; } lldiv_t;

div_t div(int numer, int denom);
ldiv_t ldiv(long numer, long denom);
lldiv_t lldiv(long long numer, long long denom);
```
- Computes algebraic quotient and remainder such that `numer = quot * denom + rem`, where `rem` has the same sign as `numer` and `|rem| < |denom|`.
- **Edge cases:** Division by zero (`denom == 0`) or `INT_MIN / -1` produces undefined behavior / hardware arithmetic trap.

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
- Dynamic memory management in freestanding libc is mediated by pluggable allocator backends (e.g. bump, slab, or buddy allocators) provided by the host environment or runtime.
- **Edge cases:**
  - `malloc(0)` returns `NULL` or a unique non-dereferenceable pointer.
  - `free(NULL)` is a safe no-op.
  - `calloc`: Returns zero-initialized memory. If `nmemb * size` overflows `size_t`, fails and returns `NULL`.
  - `realloc(NULL, size)` is equivalent to `malloc(size)`. `realloc(ptr, 0)` frees `ptr` and returns `NULL`.

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

#### Multiple Inclusion Semantics
`<assert.h>` is explicitly designed without standard `#ifndef` include guards. Each time `<assert.h>` is included, it undefines `assert` (`#undef assert`) and redefines it according to the presence or absence of the `NDEBUG` macro at that specific point in the translation unit:

```c
#undef assert

#ifdef NDEBUG
  #define assert(expr) ((void)0)
#else
  #define assert(expr) \
      ((expr) ? (void)0 : __assert_fail(#expr, __FILE__, __LINE__, __func__))
#endif
```

#### C11 vs C23 Diagnostic Interfaces
- **C11:** Supplies the `static_assert(const_expr, string_literal)` macro mapping to `_Static_assert`.
- **C23:** `static_assert` is a standard keyword supporting single-argument `static_assert(const_expr)` and two-argument `static_assert(const_expr, string_literal)`. The `assert(...)` macro in C23 permits variable macro arguments.

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

- **Startup / Entry Point:** Freestanding code compiled with `-nostdlib` contains no standard `_start` or `main` startup wrapper. Execution begins directly at the consumer platform's entry point (e.g. EFI boot service, kernel entry, or fiber entry).
- **Default Hook Fallbacks:** If a consumer does not provide custom hooks, default weak implementations in libc execute tight halt loops (`for(;;);` / CPU pause / trap instruction) upon `__libc_abort()` or assertion failure.

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

# Freestanding C Standard Library Specification

This document defines the formal interface contracts, type requirements, function signatures, and execution invariants for the freestanding C standard library (`libc`).

---

## 1. Scope & Freestanding Execution Environment

The freestanding C standard library complies with the ISO C standard (C11/C23) freestanding environment specifications while extending core facilities to satisfy the requirements of microkernel environments, embedded runtimes, and user-space green threading engines.

### 1.1 Invariants
- **Zero Host Syscalls:** Functions defined herein do not make direct host OS kernel calls. Any environmental interaction (such as heap expansion or abnormal termination) is mediated through pluggable hooks or supervisor traps.
- **Static Resolution:** All routines must resolve strictly through static linkage (`-static`); dynamic symbol resolution, GOT/PLT indirections, and shared libraries are strictly disallowed.
- **Zero Thread-Local Storage Overhead:** Core standard routines must not depend on OS-provided `_Thread_local` / `__thread` variables unless explicitly encapsulated.
- **Memory Safety & Invariants:** Memory copy and set operations must maintain alignment correctness and predictable behavior under constrained stack environments.

---

## 2. Standard Header Specifications

### 2.1 `<stddef.h>`

Defines standard primitive types, constants, and macros.

#### Types
- `size_t`: Unsigned integer type of the result of the `sizeof` operator (`unsigned long` on x86-64).
- `ptrdiff_t`: Signed integer type of the result of subtracting two pointers (`long` on x86-64).
- `max_align_t`: Type whose alignment requirement is at least as strict as that of every scalar type (`long double` or 16-byte aligned struct on x86-64).
- `wchar_t`: Integer type whose range of values can represent distinct wide codes.

#### Constants & Macros
- `NULL`: Null pointer constant represented as `((void *)0)` or `0`.
- `offsetof(type, member)`: Macro expanding to an integral constant expression of type `size_t`, representing the offset in bytes from the beginning of `type` to `member`.

---

### 2.2 `<stdint.h>`

Defines fixed-width integer types and integer boundary limits.

#### Exact-Width Integer Types
- `int8_t`, `int16_t`, `int32_t`, `int64_t`
- `uint8_t`, `uint16_t`, `uint32_t`, `uint64_t`

#### Pointer-Holding Integer Types
- `intptr_t`: Signed integer type capable of holding any pointer value without truncation (`long` on x86-64).
- `uintptr_t`: Unsigned integer type capable of holding any pointer value without truncation (`unsigned long` on x86-64).

#### Greatest-Width Integer Types
- `intmax_t`, `uintmax_t`

#### Limit Macros
- `INT8_MIN`, `INT8_MAX`, `UINT8_MAX`
- `INT16_MIN`, `INT16_MAX`, `UINT16_MAX`
- `INT32_MIN`, `INT32_MAX`, `UINT32_MAX`
- `INT64_MIN`, `INT64_MAX`, `UINT64_MAX`
- `INTPTR_MIN`, `INTPTR_MAX`, `UINTPTR_MAX`
- `SIZE_MAX`, `PTRDIFF_MIN`, `PTRDIFF_MAX`

---

### 2.3 `<string.h>` & `<memory.h>`

Defines memory manipulation and string inspection routines. `<memory.h>` is provided as an alias/subset header exposing the memory routines.

#### Memory Operations
```c
void *memcpy(void *dest, const void *src, size_t n);
```
- Copies `n` bytes from memory area `src` to memory area `dest`.
- Memory areas must not overlap. Returns `dest`.

```c
void *memmove(void *dest, const void *src, size_t n);
```
- Copies `n` bytes from memory area `src` to memory area `dest`.
- Memory areas may overlap safely. Returns `dest`.

```c
void *memset(void *s, int c, size_t n);
```
- Fills the first `n` bytes of memory area `s` with the constant byte `(unsigned char)c`. Returns `s`.

```c
int memcmp(const void *s1, const void *s2, size_t n);
```
- Compares the first `n` bytes of memory areas `s1` and `s2`.
- Returns `< 0`, `0`, or `> 0` depending on whether `s1` is less than, equal to, or greater than `s2`.

```c
void *memchr(const void *s, int c, size_t n);
```
- Locates the first occurrence of `(unsigned char)c` in the initial `n` bytes of memory area `s`.
- Returns pointer to matching byte, or `NULL` if not found.

#### String Operations
```c
size_t strlen(const char *s);
```
- Computes length of null-terminated string `s` (excluding terminating null byte `\0`).

```c
int strcmp(const char *s1, const char *s2);
```
- Compares null-terminated strings `s1` and `s2`.
- Returns `< 0`, `0`, or `> 0`.

```c
int strncmp(const char *s1, const char *s2, size_t n);
```
- Compares at most `n` bytes of strings `s1` and `s2`.
- Returns `< 0`, `0`, or `> 0`.

```c
char *strcpy(char *dest, const char *src);
```
- Copies null-terminated string `src` to `dest` (including `\0`). Returns `dest`.

```c
char *strncpy(char *dest, const char *src, size_t n);
```
- Copies at most `n` bytes of `src` to `dest`. If `src` has length `< n`, pads `dest` with trailing null bytes. Returns `dest`.

```c
char *strcat(char *dest, const char *src);
```
- Appends string `src` to `dest`, overwriting terminating `\0` in `dest`. Returns `dest`.

```c
char *strncat(char *dest, const char *src, size_t n);
```
- Appends at most `n` bytes of `src` to `dest`, followed by a terminating `\0`. Returns `dest`.

```c
char *strchr(const char *s, int c);
```
- Returns pointer to first occurrence of character `(char)c` in string `s`, or `NULL`.

```c
char *strrchr(const char *s, int c);
```
- Returns pointer to last occurrence of character `(char)c` in string `s`, or `NULL`.

```c
char *strstr(const char *haystack, const char *needle);
```
- Finds the first occurrence of substring `needle` in `haystack`. Returns pointer or `NULL`.

---

### 2.4 `<stdlib.h>`

Defines integer arithmetic, conversions, sorting/searching algorithms, and memory management interfaces.

#### Numeric Conversions
```c
int atoi(const char *nptr);
long atol(const char *nptr);
long long atoll(const char *nptr);
long strtol(const char *nptr, char **endptr, int base);
unsigned long strtoul(const char *nptr, char **endptr, int base);
long long strtoll(const char *nptr, char **endptr, int base);
unsigned long long strtoull(const char *nptr, char **endptr, int base);
```
- Parses string `nptr` according to specified radix/base (supporting bases 2 through 36, or auto-detect base 0).
- Skips leading whitespace and handles optional sign indicators.

#### Integer Arithmetic
```c
int abs(int j);
long labs(long j);
long long llabs(long long j);
```
- Computes absolute value of integer argument.

```c
typedef struct { int quot; int rem; } div_t;
typedef struct { long quot; long rem; } ldiv_t;
typedef struct { long long quot; long long rem; } lldiv_t;

div_t div(int numer, int denom);
ldiv_t ldiv(long numer, long denom);
lldiv_t lldiv(long long numer, long long denom);
```
- Computes quotient and remainder in a single operation.

#### Searching & Sorting
```c
void qsort(void *base, size_t nmemb, size_t size,
           int (*compar)(const void *, const void *));
```
- Sorts an array of `nmemb` elements of size `size` pointed to by `base`.

```c
void *bsearch(const void *key, const void *base, size_t nmemb, size_t size,
              int (*compar)(const void *, const void *));
```
- Binary search for `key` in sorted array `base`. Returns pointer to matching element or `NULL`.

#### Freestanding Memory Allocation Interface
```c
void *malloc(size_t size);
void free(void *ptr);
void *calloc(size_t nmemb, size_t size);
void *realloc(void *ptr, size_t size);
```
- Contracts for dynamic memory allocation. In freestanding environments, these functions bind to project-provided allocator backends (e.g., slab/buddy/bump allocators) without host virtual memory assumptions.

#### Process Termination & Supervisor Hooks
```c
_Noreturn void abort(void);
_Noreturn void exit(int status);
```
- `abort()`: Triggers immediate CPU trap / supervisor fault handler to halt or reset payload execution.
- `exit(status)`: Cleanly yields control back to supervisor/runtime with return code `status`.

---

### 2.5 `<assert.h>`

Defines diagnostic assertion facilities.

#### Macro Specification
```c
#ifdef NDEBUG
#define assert(expr) ((void)0)
#else
#define assert(expr) \
    ((expr) ? (void)0 : __assert_fail(#expr, __FILE__, __LINE__, __func__))
#endif
```

#### Diagnostic Hook
```c
_Noreturn void __assert_fail(const char *assertion, const char *file,
                             unsigned int line, const char *function);
```
- Dispatches failure details to serial UART / supervisor console and triggers an immediate trap / `abort()`.

---

## 3. Linkage & ABI Requirements

1. **Target Architecture:** x86-64 (AMD64) System V ABI calling convention (arguments in `%rdi`, `%rsi`, `%rdx`, `%rcx`, `%r8`, `%r9`, return value in `%rax`/`%rdx`).
2. **Stack Alignment:** Functions must preserve 16-byte stack alignment prior to `call` instructions as dictated by the x86-64 ABI.
3. **Red Zone Behavior:**
   - In Ring 0 (kernel/storage), code must be compiled with `-mno-red-zone`.
   - In Ring 3 (user payloads), red-zone utilization conforms to standard SysV ABI unless disabled by configuration.
4. **Static Toolchain Compliance:** No undefined references to dynamic libc symbols (`dlopen`, `dlsym`, `__libc_start_main`, etc.).

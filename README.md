# Freestanding C Standard Library (`libc`) Specification

A minimal, zero-dependency, freestanding C standard library specification repository engineered for low-level operating systems, bare-metal microkernels, and embedded runtime environments.

---

## 1. Overview & Architectural Role

This repository provides formal interface contracts, standard header definitions, and freestanding C standard library specifications for low-level operating environments and standalone freestanding x86-64 projects.

### Core Objectives:
- **Zero Host Dependencies:** Completely decoupled from host operating system headers, dynamic loaders, and standard hosted runtimes.
- **Freestanding Interface Clarity:** Formally defines mandated ISO C (C11/C23) freestanding headers alongside supported hosted extensions and compatibility interfaces.
- **Static Linking Exclusivity:** All specifications mandate static compilation and resolution (`-static`).
- **Ring Separation Support:** Defines explicit compilation profiles for Ring 0 supervisor/kernel components and Ring 3 user-space application payloads.

---

## 2. Compilation Profiles & Toolchain Flags

Consumers linking against this library must adhere to static, freestanding compilation constraints. The library defines two distinct execution profiles:

### 2.1 Ring 0 (Kernel & Storage Profile)
When compiled for or linked into Ring 0 kernel, HAL, or journaled storage objects, compiler flags must prevent stack pollution, red-zone corruption during asynchronous hardware interrupts, and reliance on host runtime mechanisms:

```bash
-ffreestanding -nostdlib -fno-builtin -fno-stack-protector -mno-red-zone -static
```

- `-ffreestanding`: Directs the compiler not to assume standard hosted library builtins or runtime startup routines.
- `-nostdlib`: Prevents linking against standard system startup files (`crt0.o`, etc.) or host standard libraries.
- `-fno-builtin`: Disables implicit substitution of standard functions with host compiler intrinsics.
- `-fno-stack-protector`: Eliminates dependency on host TLS stack canary runtime symbols (e.g., `__stack_chk_guard`, `__stack_chk_fail`).
- `-mno-red-zone`: Disables the 128-byte x86-64 red zone below `%rsp`, preventing asynchronous interrupt handlers from corrupting local leaf-function stack frames.
- `-static`: Eliminates all dynamic linking and runtime symbol relocations.

### 2.2 Ring 3 (User Payload & Concurrency Profile)
When compiled for standalone ELF application binaries or user-space concurrency payloads:

```bash
-ffreestanding -nostdlib -fno-stack-protector -static
```

- `-fno-stack-protector` is explicitly included for toolchains where stack protection is enabled by default, avoiding unresolved references to canary symbols when no thread-local storage or runtime guard handlers exist.
- System interaction is mediated exclusively via explicit supervisor traps and consumer-provided hooks rather than implicit host syscalls.

---

## 3. Supported Standard Headers & Scope Taxonomy

The specification categorizes headers into mandated ISO freestanding headers, provided hosted facilities, and non-standard extensions:

| Header | Standard Category | Description & Supported Scope |
| :--- | :--- | :--- |
| `<stddef.h>` | ISO Mandated (Freestanding C11) | Fundamental types (`size_t`, `ptrdiff_t`, `max_align_t`, `wchar_t`), constants (`NULL`), and macros (`offsetof`). |
| `<stdint.h>` | ISO Mandated (Freestanding C11) | Exact-width, least-width, fast-width, pointer, and maximum-width integer types, limit macros, and constant macros. |
| `<string.h>` | Mandated in C23 Freestanding / Hosted in C11 | Memory manipulation (`memcpy`, `memmove`, `memset`, `memcmp`, `memchr`) and string operations (`strlen`, `strcmp`, `strcpy`, `strcat`, etc.). |
| `<stdlib.h>` | ISO Hosted (Selected Facilities Provided) | Integer arithmetic (`abs`, `div`), numeric conversions (`atoi`, `strtol`), searching/sorting (`qsort`, `bsearch`), allocator contracts (`malloc`, `free`), and exit hooks (`abort`, `exit`). |
| `<assert.h>` | ISO Hosted (Diagnostic Facility Provided) | Diagnostic assertion macros (`assert`, `static_assert`) with redefinition on multiple inclusion and pluggable failure hooks. |
| `<memory.h>` | Non-Standard Extension (Compatibility) | Compatibility header aliasing standard memory manipulation routines from `<string.h>`. |

Detailed function signatures, behavioral contracts, edge-case semantics, and runtime hook definitions are documented in [`SPECIFICATION.md`](SPECIFICATION.md).

---

## 4. Repository Structure & Boundary Rules

```text
libc/
├── LICENSE                 # Submodule license (MIT)
├── README.md               # Architecture, compilation profiles & taxonomy overview
└── SPECIFICATION.md        # API contracts, signatures, and freestanding invariants
```

### Upstream Isolation Policy:
- Modifications to standard library interfaces and implementations belong upstream in this repository and are integrated into downstream projects via pinned Git submodule references (`libs/libc`).
- Specifications define pure software contracts; target-specific hardware details (e.g., UART addresses, MMIO registers) are decoupled from the libc interface and handled by consumer runtime hooks.

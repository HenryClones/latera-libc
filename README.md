# Freestanding C Standard Library (`libc`)

A minimal, zero-dependency, freestanding C standard library specification and implementation repository engineered for low-level operating systems, bare-metal microkernels, and embedded runtime environments.

---

## 1. Overview & Architectural Role

This repository provides the formal interface contracts, standard headers, and freestanding C standard library specifications for the **Latera** operating environment and standalone freestanding x86-64 projects.

### Core Objectives:
- **Zero Host Dependencies:** Completely independent of host operating system headers, dynamic loaders, and standard system runtimes.
- **Freestanding Purity:** Conforms to C11/C23 freestanding environment specifications with 100% static linking (`-static`).
- **Ring Separation Support:** Defines explicit compilation profiles for Ring 0 supervisor/kernel components and Ring 3 user-space application payloads.
- **Concurrency Integration:** Supplies fundamental standard library primitives for Latera's lightweight green threads concurrency engine (`src/concurrency/green_threads.c`).

---

## 2. Compilation Profiles & Toolchain Flags

All consumers linking against this library must adhere to static, freestanding compilation constraints. The library supports two distinct execution profiles:

### 2.1 Ring 0 (Kernel & Storage Profile)
When compiled for or linked into Ring 0 kernel, HAL, or journaled storage objects, compiler flags must prevent stack pollution, red-zone corruption during asynchronous hardware interrupts, and reliance on host runtime mechanisms:

```bash
-ffreestanding -nostdlib -fno-builtin -fno-stack-protector -mno-red-zone -static
```

- `-ffreestanding`: Directs the compiler not to assume standard library builtins or a hosted runtime environment.
- `-nostdlib`: Prevents linking against standard system startup files (`crt0.o`, etc.) or host libraries.
- `-fno-builtin`: Disables implicit substitution of standard functions with host compiler intrinsics.
- `-fno-stack-protector`: Eliminates dependency on host TLS stack canary runtime symbols (e.g., `__stack_chk_guard`).
- `-mno-red-zone`: Disables the 128-byte x86-64 red zone below `%rsp`, preventing asynchronous interrupt handlers from corrupting local leaf-function stack frames.
- `-static`: Eliminates all dynamic linking and runtime symbol relocations.

### 2.2 Ring 3 (User Payload & Concurrency Profile)
When compiled for standalone ELF application binaries or test payloads running under the Latera green threads concurrency runtime:

```bash
-ffreestanding -nostdlib -static
```

- Guarantees clean isolation from host facilities while permitting standard register utilization conventions.
- System interaction is mediated exclusively via explicit Latera supervisor interfaces rather than implicit host syscalls.

---

## 3. Supported Standard Headers & Scope

The freestanding standard library specification defines explicit interface contracts for the following core headers:

| Header | Description & Scope |
| :--- | :--- |
| `<stddef.h>` | Standard type definitions (`size_t`, `ptrdiff_t`, `NULL`, `offsetof`, `max_align_t`). |
| `<stdint.h>` | Exact-width integer types (`int8_t`..`int64_t`, `uint8_t`..`uint64_t`, `intptr_t`, `uintptr_t`) and limits. |
| `<string.h>` | Memory manipulation (`memcpy`, `memmove`, `memset`, `memcmp`, `memchr`) and string operations (`strlen`, `strcmp`, `strcpy`, `strcat`, etc.). |
| `<memory.h>` | Compatibility header aliasing standard memory manipulation routines. |
| `<stdlib.h>` | Numeric conversions (`atoi`, `strtol`, etc.), integer arithmetic (`abs`, `div`), sorting/searching (`qsort`, `bsearch`), and allocator/supervisor exit contracts (`malloc`, `free`, `abort`). |
| `<assert.h>` | Freestanding diagnostic assertion macro (`assert`) backed by supervisor trap hooks. |

Detailed function signatures, behavioral specifications, and memory invariants are documented in [`SPECIFICATION.md`](SPECIFICATION.md).

---

## 4. Repository Structure & Boundary Rules

```text
libc/
├── LICENSE                 # Submodule license (MIT)
├── README.md               # Architecture, compilation profiles & usage overview
└── SPECIFICATION.md        # API contracts, signatures, and freestanding invariants
```

### Upstream Isolation Policy:
- Modifications to standard library interfaces and implementations belong upstream in this repository and are integrated into downstream projects (such as Latera) via pinned Git submodule references (`libs/libc`).
- Consumers must not introduce target-specific kernel assembly or host-dependent extensions into this library boundary.

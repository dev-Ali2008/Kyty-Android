<p align="center">
    <a href="https://github.com/dev-Ali2008/Kyty-Android/blob/7aadd2e477e32baff0a4cb678fac84f03976d316/icon_App.pnp">
        <img height="150px" src="https://github.com/dev-Ali2008/Kyty-Android/blob/7aadd2e477e32baff0a4cb678fac84f03976d316/icon_App.png" />
    </a>
</p>


# Kyty - PS4 & PS5 Emulator for Android

Android port of the [Kyty](https://github.com/InoriRus/Kyty) PS4/PS5 emulator. x86_64 interpreter on ARM64, Vulkan 1.1 rendering, AMD GCN to SPIR-V shader translation, Kotlin UI talking to C++ core through JNI.

> Brother This is a real emulator — it runs actual PS5 ELF binaries on your phone Not a video player - not streaming, not pre-recorded And **NOT SCAMMER ?!** The interpreter decodes x86_64 instructions one by one on the ARM64 CPU HLE syscalls map to real Linux kernel calls and Vulkan renders frames through your GPU driver... How it all works is explained in [How_ITS_work?.md](https://github.com/dev-Ali2008/Kyty-Android/blob/main/How_ITS_work%3F.md).

## Repository Policy

**Repository Purpose:** To safeguard the codebase and maintain proprietary performance enhancements (Kyty Android is closed-source This official) repository does not contain the emulator's source code Instead, it serves as the official version archive and primary distribution platform for releasing APK builds managing compatibility issues and sharing user guides...

## Status

Still early days. Core subsystems are up and running — ELF loading, interpreter execution, HLE dispatch, basic Vulkan pipeline. Working through game-specific issues as they come up.

### What's working
- ELF loading (PS4/PS5 format)
- x86_64 instruction decoding and execution on ARM64
- ~200+ opcodes so far:
  - Standard ALU, MOV, PUSH/POP, CALL/RET, JMP/Jcc
  - Bit rotation: ROL/ROR/RCL/RCR (1 and CL variants)
  - PUSHFQ/POPFQ for full RFLAGS register save/restore
  - CPUID with PS4/PS5-compatible feature bits
  - POPCNT through BMI1 detection
  - String ops: MOVS, CMPS, STOS, LODS, SCAS with REP prefix
  - Full ModR/M + SIB + displacement addressing
  - SSE/SSE2 packed integer and double-precision
  - MFENCE/LFENCE/SFENCE memory fences
  - WAIT and FPU control
  - FS/GS segment override for TLS and stack canary
- PLT native dispatch:
  - Native function pointer slots for HLE calls
  - `GetMem` reads with null-check safety
  - SYSTEM_RESERVED range excluded from native detection
  - Resolved addresses stored during ELF relocation
- Vulkan 1.1 pipeline with runtime SPIR-V shader assembly
- HLE syscall dispatch — x86_64 SysV ABI mapped to ARM64 AAPCS64
- 70+ syscalls:
  - Memory: mmap, munmap, mprotect, mquery, madvise
  - Threads: thr_new, thr_exit, mutex, condvar, rwlock
  - Dynlib: load_prx, get_info, get_list, prepare_dlclose, process_needed_and_relocate
  - Time: clock_gettime, nanosleep, gettimeofday, select
  - I/O: open, close, read, write, writev, ioctl
  - Filesystem: mkdir, rmdir, getdents, stat, unlink
  - Signal: sigaction, sigprocmask, sigaltstack, sigtimedwait
  - Misc: sysarch, tls_setup, cpuset, dynlib_do_copy_relocations
- Kotlin UI with file picker, ROM selection, and emulator controls
- Interpreter self-test suite covering ALU, memory, branches, SSE, stack, stress

### Still working on
- PS4/PS5 HLE library integration (SysV, Memory, Display, etc.)
- GPU command processing (PM4 draw dispatch)
- Audio subsystem
- Controller/input mapping

### Test

#### PS5
<img src="https://github.com/dev-Ali2008/Kyty-Android/blob/08613effa7fc45dcc063d489d4d9d2226a63f3fe/Vulkan%201.1.png" width="800">

**The kyty ps5 homebrews with glitches but we edit it Core to Native - other codes CPP full**


## Architecture

```
Kotlin UI (EmulatorActivity)
    |
    | JNI (kyty_jni.cpp)
    |
    v
Kyty C++ Core
    |
    +-- ELF Loader (RuntimeLinker)
    |   +-- Segment relocation
    |   +-- PLT allocation (ReadWrite native pointer slots)
    |   +-- DT_NEEDED parsing (no auto-load yet)
    +-- x86_64 CPU Interpreter (x86_cpu.cpp)
    |   +-- ~200+ opcodes (ALU, SIMD, string, segment, FPU)
    |   +-- FS/GS segment override (TLS access)
    |   +-- PLT native dispatch (GetMem + DispatchNativeCall)
    |   +-- Thread-local state per pthread
    +-- HLE System Calls (DispatchNativeCall)
    |   +-- x86_64 SysV ABI → ARM64 AAPCS64
    |   +-- 70+ syscalls (memory, threads, dynlib, I/O, signal)
    +-- Vulkan Renderer (spirv-tools runtime assembly)
    +-- AMD GCN -> SPIR-V Shader Translation
    +-- Lua Scripting Engine
    |
    v
Android NDK / Vulkan 1.1 / ARM64 Native
```

## How it actually works

People keep asking how this runs on Android. Here's the short version:

1. **ELF Loading** — The PS5 game binary gets `mmap`'d into the process at the same virtual addresses it expects. The interpreter sees the code at the same address the PS5 kernel would.

2. **x86_64 Interpretation** — Every instruction is decoded and executed by a big switch/case loop running on the ARM64 CPU. No translation layer, no recompilation — just reading x86_64 bytes and doing what they say.

3. **HLE Syscalls** — When the game tries to call a PS5 kernel function through the `syscall` instruction, we intercept it. `sys_mmap` becomes Linux `mmap`, `sys_pthread_create` becomes `pthread_create`. The game gets real memory allocation and real threads.

4. **PLT Dispatch** — Library function calls go through a table of function pointers that we fill in during ELF loading with pointers to our native implementations.

5. **Vulkan Rendering** — GPU commands come out as AMD GCN PM4 packets. We parse those and translate them to Vulkan 1.1 calls that the Android GPU driver actually executes.

There's a longer explanation with code examples in [How_ITS_work?.md](https://github.com/dev-Ali2008/Kyty-Android/blob/main/How_ITS_work%3F.md).

## Key Features

### x86_64 CPU Interpreter

Android runs ARM64, PS5 games are x86_64. Can't run the instructions natively. So we interpret them — read each byte, figure out what it means, execute it.

- ModR/M + SIB + displacement addressing fully decoded
- ~200+ opcodes: MOV, PUSH/POP, CALL/RET, JMP/Jcc, ALU, strings, segments
- SSE/SSE2: MOVSS/MOVSD, ADDSD/SUBSD/MULSD/DIVSD, CVTTSS2SI, packed ops
- CPUID, POPCNT, FENCE (MFENCE/LFENCE/SFENCE)
- FS/GS segment override for TLS — game reads GS:0x28 for stack canary
- PLT dispatch for HLE calls
- SYSCALL interception
- 64-bit address space with direct pointer access
- 16MB stack per interpreter thread

### PLT Native Dispatch

When the interpreter hits a `CALL r/m64` or `CALL rel32` into the PLT range (`0x800000000`–`0x900000000`), it reads the native function pointer from that slot and calls it directly:

- Slots filled during ELF relocation with resolved native addresses
- `GetMem` with null-check — unresolved entries log and return 0 instead of crashing
- x86_64 SysV ABI (RDI/RSI/RDX/RCX/R8/R9) mapped to ARM64 AAPCS64 (X0–X5)
- Up to 6 integer arguments
- Return value propagated back to x86_64 RAX

### Vulkan 1.1 Rendering

Runtime SPIR-V shader assembly using spirv-tools — no offline shader compiler needed:

- Instance, Device, and Swapchain management
- Android Surface integration
- Graphics pipeline with triangle test for verification
- Vertex and fragment SPIR-V shaders compiled at runtime via `spvTextToBinary()`

### Kotlin UI
- ROM/ELF file picker with storage access framework
- Emulator lifecycle (init, load, run, stop)
- Surface view for Vulkan output
- Per-test PASS/FAIL display

## Testing

Self-test suite with 6 categories:

| Test | Description | Expected |
|------|-------------|----------|
| ALU + HLE | Basic arithmetic + HLE dispatch (add, mul, sub, and, or, xor, shl, shr) | 1432 |
| Memory | Byte/word/dword/qword read/write round-trip | 0x9ABCDEF0 |
| Conditional Jumps | Jcc loop counting to 100 | 100 |
| SSE Floats | MOVSS/ADDSS/MULSS/DIVSS/CVTTSS2SI | 4375 |
| Stack | PUSH/POP LIFO ordering | 4400 |
| Stress | 100K iteration loop with accumulator | varies |

Results show in-app with per-test PASS/FAIL status.

## Third-Party Libraries

| Library | Purpose |
|---------|---------|
| [Vulkan SDK](https://www.vulkan.org/) | Graphics API |
| [SPIR-V Tools](https://github.com/KhronosGroup/SPIRV-Tools) | Runtime shader compilation |
| [Lua](https://www.lua.org/) | Scripting engine |
| [SDL2](https://www.libsdl.org/) | Window/input management |
| [Zstandard](https://facebook.github.io/zstd/) | Compression |
| [LZMA SDK](https://www.7-zip.org/sdk.html) | Compression |
| [SQLite](https://www.sqlite.org/) | Database |
| [cpuinfo](https://github.com/pytorch/cpuinfo) | CPU feature detection |
| [easy_profiler](https://github.com/yse/easy_profiler) | Performance profiling |
| [xxHash](https://github.com/Cyan4973/xxHash) | Fast hashing |
| [STB](https://github.com/nothings/stb) | Image loading |
| [magic_enum](https://github.com/Neargye/magic_enum) | Enum reflection |
| [miniz](https://github.com/richgel999/miniz) | Zlib-compatible compression |
| [Rijndael](https://github.com/DavyLandman/Rijndael) | AES encryption |

## Credits

- **[InoriRus](https://github.com/InoriRus/Kyty)** - Original Kyty PS4/PS5 emulator
- **Android Port** - Native Android integration with x86_64 interpreter

## License

This project inherits the license of the original [Kyty emulator](https://github.com/InoriRus/Kyty). See the original repository for licensing details.

## Disclaimer

This software is for educational and research purposes only. It is not affiliated with or endorsed by Sony Interactive Entertainment. PlayStation is a registered trademark of Sony Interactive Entertainment.

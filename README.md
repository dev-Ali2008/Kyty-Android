# Kyty - PS4 & PS5 Emulator for Android

A native Android port of the [Kyty](https://github.com/InoriRus/Kyty) PS4/PS5 HLE (High-Level Emulation) emulator, featuring a full x86_64 CPU interpreter, Vulkan 1.1 rendering pipeline, AMD GCN shader translation, and a complete JNI bridge between Kotlin UI and C++ emulation core.

## Repository Policy

**Repository Purpose:** To safeguard the codebase and maintain proprietary performance enhancements (Kyty Android is closed-source This official) repository does not contain the emulator's source code Instead, it serves as the official version archive and primary distribution platform for releasing APK builds managing compatibility issues and sharing user guides... 

## Status

**Early Development** - The emulator is in active development Core systems are being integrated and tested on Android ARM64 devices.

### What Works
- ELF loading (PS4/PS5 format)
- x86_64 CPU instruction decoding and execution (ARM64 interpreter)
- Vulkan 1.1 rendering pipeline with runtime SPIR-V shader assembly
- HLE system call dispatch (x86_64 SysV ABI to ARM64 AAPCS64)
- Kotlin UI with file picker, ROM selection, and emulator controls
- Interpreter self-test suite (ALU, memory, branches, SSE, stack, stress)

### In Progress
- PS4/PS5 HLE library integration (SysV, Memory, Display, etc.)
- GPU command processing
- Audio subsystem
- Controller/input mapping

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
    +-- x86_64 CPU Interpreter (x86_cpu.cpp)
    +-- HLE System Calls (DispatchNativeCall)
    +-- Vulkan Renderer (spirv-tools runtime assembly)
    +-- AMD GCN -> SPIR-V Shader Translation
    +-- Lua Scripting Engine
    |
    v
Android NDK / Vulkan 1.1 / ARM64 Native
```

## Key Features

### x86_64 CPU Interpreter
Since Android devices use ARM64 processors, PS4/PS5 x86_64 machine code cannot execute natively. The interpreter decodes and executes x86_64 instructions at runtime:

- Full ModR/M + SIB addressing mode decoding
- ~150+ x86_64 opcodes (MOV, PUSH/POP, CALL/RET, JMP/Jcc, ALU ops)
- SSE instruction support (MOVSS/MOVSD, ADDSS/SUBSS/MULSS/DIVSS, CVT*)
- SYSCALL interception for HLE dispatch
- 64-bit memory model with direct pointer translation
- 16MB interpreter stack

### HLE Call Interception
When the interpreter encounters a `CALL r/m64` instruction targeting a PLT stub (address >= 0x800000000000), it intercepts the call and dispatches it as a native ARM64 function call:

- Translates x86_64 SysV ABI (RDI/RSI/RDX/RCX/R8/R9) to ARM64 AAPCS64 (X0-X5)
- Supports up to 6 integer arguments
- Automatic return value propagation back to x86_64 RAX

### Vulkan 1.1 Rendering
Runtime SPIR-V shader assembly using spirv-tools (no offline shader compiler needed):

- Instance, Device, and Swapchain management
- Android Surface integration
- Graphics pipeline with hardcoded triangle test
- Vertex and fragment SPIR-V shaders compiled at runtime via `spvTextToBinary()`

### Kotlin UI
- ROM/ELF file picker with storage access framework
- Emulator lifecycle management (init, load, run, stop)
- Surface view for Vulkan rendering output
- Per-test result display with detailed PASS/FAIL reporting

## Testing

The interpreter includes a comprehensive self-test suite with 6 categories:

| Test | Description | Expected |
|------|-------------|----------|
| ALU + HLE | Basic arithmetic + HLE dispatch (add, mul, sub, and, or, xor, shl, shr) | 1432 |
| Memory | Byte/word/dword/qword read/write round-trip | 0x9ABCDEF0 |
| Conditional Jumps | Jcc loop counting to 100 | 100 |
| SSE Floats | MOVSS/ADDSS/MULSS/DIVSS/CVTTSS2SI | 4375 |
| Stack | PUSH/POP LIFO ordering | 4400 |
| Stress | 100K iteration loop with accumulator | varies |

Test results are displayed in-app with per-test PASS/FAIL status.

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

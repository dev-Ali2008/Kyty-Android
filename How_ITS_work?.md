# How Kyty Runs PS5 Games on Android

> I keep seeing questions about how this works, so here's the full breakdown. Actual code paths, actual memory layouts, no hand-waving.

---

## The short version

PS5 games are compiled for x86_64 — a completely different CPU architecture than what's in your Android phone (ARM64). There's no magic here, it's the same approach every other emulator uses: decode each instruction and execute it.

Three things make it work:

1. **x86_64 interpreter** — reads PS5 machine code byte by byte and executes it on ARM64
2. **HLE syscalls** — when the game asks the PS5 kernel to do something, we intercept it and do it with Linux instead
3. **Vulkan rendering** — PS5 GPU commands get translated to Vulkan calls that your Adreno/Mali driver understands

PCSX2 - Yuzu -RPCS3 all started this way.

---

## Loading the ELF

A PS5 game comes as a PKG file. Inside that is a PS5 ELF — same format as Linux ELFs but compiled for x86_64 with PS5-specific sections.

```
What actually happens:
  1. Read ELF header, confirm e_machine == EM_X86_64
  2. For each PT_LOAD segment:
     mmap(PROT_READ|PROT_WRITE, MAP_FIXED) at the segment's vaddr
     So .text ends up at 0x900000000, .data at 0x907000000, etc.
  3. Process relocations (R_X86_64_RELATIVE, R_X86_64_64, ...)
  4. Fill PLT slots with native function pointers
  5. Start at e_entry (usually 0x900000070)
```

The key thing: the ELF's virtual addresses are real addresses in our process. We use `mmap(MAP_FIXED)` so the code lives exactly where it expects to. When the game reads address `0x900000070` that's where the actual bytes are. No address translation needed.

---

## The interpreter

PS5 code is x86_64. Android's CPU is ARM64. They're completely incompatible instruction sets. So we decode each x86_64 instruction and execute it manually:

```cpp
// What the main loop actually looks like (simplified)
while (running) {
    uint8_t byte = GetMem(state.rip, 1);
    
    switch (byte) {
        case 0x48:  // REX.W prefix
            state.rex_w = true;
            state.rip++;
            break;
        case 0x89:  // MOV r/m64, r64
            decode_modrm();
            write_operand(decode_ea_from(modrm), state.reg64(r));
            state.rip += inst_len;
            break;
        case 0xE8:  // CALL rel32
            int32_t rel = read32(state.rip + 1);
            push64(state.rip + 5);
            state.rip += 5 + rel;
            break;
        // ... ~200 more cases
    }
}
```

It's a big switch statement. Boring, but correct by construction — every instruction is handled explicitly.

### What's implemented

| Category | Examples | Rough count |
|----------|----------|-------------|
| Data movement | MOV, LEA, XCHG, MOVSX, MOVZX | ~30 |
| Arithmetic | ADD, SUB, MUL, DIV, INC, DEC, NEG, CMP | ~25 |
| Bitwise | AND, OR, XOR, NOT, SHL, SHR, SAR, ROL, ROR, RCL, RCR | ~20 |
| Control flow | JMP, CALL, RET, Jcc (all conditions), LOOP | ~35 |
| Stack | PUSH, POP, PUSHFQ, POPFQ, ENTER, LEAVE | ~10 |
| String ops | MOVS, CMPS, STOS, LODS, SCAS with REP prefix | ~15 |
| SSE/SSE2 | MOVSS/SD, ADD/SUB/MUL/DIV, CVT, COMIS, packed ops | ~50 |
| System | CPUID, SYSCALL, INT, NOP, HLT, PAUSE, UD2 | ~15 |
| Segment | FS/GS override for TLS | 3 |
| Misc | WAIT, FENCE (MFENCE/LFENCE/SFENCE), POPCNT | ~10 |

### Why not JIT?

JIT would be faster — translate x86_64 to ARM64 native code at runtime. But:

- Needs W^X memory, Android doesn't like that
- x86_64 and ARM64 have different memory ordering guarantees
- Way more complex to get right
- Interpreter is fine for getting things working first

---

## HLE — intercepting syscalls

PS5 games don't talk to hardware directly. They call kernel functions through the `syscall` instruction:

```
sys_mmap()            allocate memory
sys_pthread_create()  spawn a thread
sys_dynlib_load_prx() load a shared library
sys_ioctl()           talk to a device
```

On a real PS5, `syscall` goes into the FreeBSD-based kernel. We catch it in the interpreter:

```cpp
// When we hit 0F 05 (SYSCALL instruction):
if (opcode == 0x0F && next == 0x05) {
    uint64_t num = state.rax;
    
    switch (num) {
        case 0x2000001:  // sys_exit
            running = false;
            break;
        case 0x20000A4:  // sys_mmap
            state.rax = host_mmap(state.rdi, state.rsi, ...);
            break;
        case 0x2000153:  // sys_dynlib_load_prx
            state.rax = fake_load_prx(state.rdi);
            break;
        // ... 70+ more
    }
    
    state.rip += 2;
}
```

### ABI translation

x86_64 SysV ABI passes arguments in RDI, RSI, RDX, RCX, R8, R9.

ARM64 AAPCS64 uses X0, X1, X2, X3, X4, X5.

We map between them:

```
RDI → X0    RSI → X1    RDX → X2
RCX → X3    R8  → X4    R9  → X5
X0  → RAX (return)
```

### What's actually implemented vs stubbed

| | What | Notes |
|--|------|-------|
| Working | Memory (mmap/munmap/mprotect/madvise) | Straight Linux syscall passthrough |
| Working | Threads (thr_new/mutex/condvar/rwlock) | Maps to pthread primitives |
| Working | I/O (open/close/read/write/ioctl) | Linux file ops |
| Working | Time (clock_gettime/nanosleep) | Linux clock |
| Stubbed | Dynlib (load_prx/get_info/list) | Return fake handles — no PS5 shared lib loading yet |
| Stubbed | Audio | No audio pipeline |
| Stubbed | Controller | No input mapping |

---

## PLT — how library calls get dispatched

When the game calls a PS5 library function (like `sce::Gnm::submitAndFlipCommandBuffers`), it goes through a PLT — a table of function pointers. We fill these in with pointers to our native implementations during ELF loading:

```
Game code:          CALL [0x800000020]     ← PLT slot #1
                         ↓
Interpreter reads:   target = GetMem(0x800000020, 8)
                         ↓
Slot contains:      native function pointer (e.g., sceKernelExitProcess)
                         ↓
DispatchNativeCall:  Map x86_64 args → ARM64 args → call it
                         ↓
Return:             Written back to x86_64 RAX
```

The PLT lives in `SYSTEM_RESERVED` range (`0x800000000`–`0x900000000`), allocated as `ReadWrite`. If a slot is zero (unresolved), we log an error and return 0 instead of crashing.

---

## Vulkan rendering

PS5 games send GPU commands as AMD GCN PM4 packets. We parse those and convert to Vulkan:

```
Game writes GPU commands
    ↓
GraphicsRun() reads PM4 packets
    ↓
GraphicsTranslate() turns them into Vulkan calls
    ↓
Vulkan driver (Adreno/Mali/etc.) does the actual rendering
    ↓
Frames show up via VkSwapchainKHR → ANativeWindow → SurfaceView
```

### Shaders

PS5 shaders are AMD GCN ISA — GPU machine code specific to AMD's architecture. We translate them:

```
PS5 GCN binary
    ↓ (decode)
IR (intermediate form)
    ↓ (code gen)
SPIR-V text
    ↓ (spirv-tools spvTextToBinary)
SPIR-V binary
    ↓ (Vulkan)
Shader module in the graphics pipeline
```

Same `spvTextToBinary()` API from the Vulkan SDK, just running at runtime on the device instead of offline.

---

## Memory layout

The Android process has a 64-bit virtual address space. We lay out PS5 memory like this:

```
0x0000000000 — 0x00FFFFFFFF : [unused]
0x0100000000 — 0x01FFFFFFFF : [Android heap/mmap]
0x0200000000 — 0x07FFFFFFFF : [interpreter stacks, one per thread]
0x0800000000 — 0x08FFFFFFFF : [PLT tables with native pointers]
0x0900000000 — 0x09FFFFFFFF : [PS5 ELF code + data]
0x0A00000000 — 0x0AFFFFFFFF : [PS5 shared libraries]
0x0B00000000 — 0x0BFFFFFFFF : [PS5 heap, mmap regions]
```

Everything's `mmap(MAP_FIXED)` so guest addresses = host addresses. `GetMem(guest_addr)` just returns a pointer.

---

## Threads

PS5 games are multi-threaded. Each PS5 thread maps to a real Linux pthread:

```
PS5 game:                    Kyty:
sys_pthread_create()    →   pthread_create() + new X86Cpu
    ↓                           ↓
Thread runs x86_64         ARM64 thread runs
via interpreter            interpreter loop for that thread
    ↓                           ↓
Has own:                  Has own:
  CpuState (rax, rip..)    X86Cpu instance (4MB stack)
  GS base (TLS ptr)        mmap'd stack memory
```

FS/GS segment overrides handle thread-local storage. GS base points to the thread's stack area — that's where TLS data lives. Stack canary lives at GS:0x28.

---

## Performance bottlenecks

| What | How bad | What we can do about it |
|------|---------|------------------------|
| Interpreter overhead | 10-50x slower than native | JIT later, maybe |
| No dynlib loading | Games can't load PS5 .so files | Stubbed for now |
| No GNM/GNMX HLE | GPU commands not fully parsed | Vulkan pipeline exists, just needs more coverage |
| Adreno format gaps | No BC3_SRGB, no D32_SFLOAT_S8_UINT | Software decode, fallback formats |
| Runtime shader compile | SPIR-V assembly is slow | Cache compiled shaders |

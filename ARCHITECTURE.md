# OWSA Architecture

## Purpose

Open Windows Subsystem Android (OWSA) is an Android runtime designed for Windows. Its intended platform is:

```text
Windows x86_64
└── Hyper-V
    └── Android x86_64 guest
        ├── ART executes classes.dex as x86_64 guest code
        ├── x86_64 ELF libraries execute directly
        └── arm64-v8a ELF libraries execute through OWSA Native Bridge
```

The project does **not** run a full ARM64 Android guest on an x86_64 PC. It uses AOSP Native Bridge as the integration boundary for foreign-ISA native libraries.

## Component map

```text
Windows host
├── Runtime orchestrator (future)
│   ├── VM lifecycle: start, ready, idle, save, stop
│   └── diagnostics collection
└── Hyper-V VM
    └── Android x86_64 guest
        ├── AOSP ART / JNI / package manager
        ├── AOSP libnativebridge
        ├── Android linker namespaces
        └── OWSA Native Bridge
            ├── adapter        NativeBridgeItf and ABI policy
            ├── loader         ARM64 ELF mapping, symbols, relocations
            ├── interpreter    reference ARM64 execution engine
            ├── jit            ARM64 IR to x86_64 code generation
            ├── runtime        JNI trampolines, threads, TLS, signals
            └── diagnostics    structured failures and compatibility state
```

## Dependency direction

Code may depend only toward lower-level layers:

```text
adapter → loader / runtime / diagnostics
loader  → runtime / diagnostics
interpreter → runtime / diagnostics
jit     → interpreter IR contracts / runtime / diagnostics
runtime → diagnostics
compatibility → diagnostics schema only
```

Forbidden dependencies:

- `loader` must not call `jit`.
- `interpreter` must not depend on Android UI, Hyper-V management, or package-specific workarounds.
- `jit` must not alter ARM64 semantics independently of the interpreter contract.
- Package-specific compatibility rules must not leak into decoder or loader core logic.

## Boundary contracts

### Android / bridge boundary

The bridge implements the Native Bridge interface expected by the selected AOSP revision. The adapter owns ABI/path support decisions, library loading entry points, trampolines, and namespace integration.

### Foreign ABI boundary

ARM64 code obeys AAPCS64. Host-generated code obeys x86_64 Android ABI. Every crossing must have an explicit trampoline and tests for arguments, return values, stack alignment, object references, thread attachment, and failure propagation.

### Execution boundary

The interpreter is authoritative for supported ARM64 instruction semantics. JIT is an optimization only; it must fall back safely and must be checked against the interpreter in verification mode.

### Security boundary

APK-native input is untrusted. ELF parsing validates all externally controlled offsets, sizes, alignments, symbol indices, and relocation records. Generated code pages follow W^X. Unsupported behavior fails closed with diagnostics.

## Current stage

The active scope is R0/R1: establish a reproducible x86_64 Android test environment and prove that a controlled `arm64-v8a/libdemo.so` load and JNI-symbol request reaches `libowsa_nativebridge.so`. See the active execution plan.

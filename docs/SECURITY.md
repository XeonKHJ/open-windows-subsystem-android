# Security Requirements

## Threat model

The project loads and executes native libraries supplied by APKs. Treat ELF metadata, dynamic-link data, relocation tables, symbol names, generated execution paths, and package-level compatibility metadata as untrusted input.

## Required controls

### ELF loader

- Validate ELF magic, class, endianness, machine, header sizes, program-header bounds, dynamic-table bounds, string-table bounds, symbol indices, relocation targets, and alignment.
- Reject integer overflow, out-of-range offsets, overlapping invalid mappings, unsupported relocation types, and malformed dependency graphs.
- Do not use host linker behavior as an implicit parser for foreign ARM64 ELF.

### Generated code

- Enforce W^X: code pages are writable only while being generated and executable only after finalization.
- Do not maintain RWX mappings.
- Bounds-check code-cache writes and metadata.
- Keep JIT verification and interpreter fallback available in development builds.

### Runtime boundaries

- Validate all cross-ABI trampoline arguments, stack alignment, thread state, JNI references, and error returns.
- Keep package-specific workarounds declarative and reviewable; do not inject opaque code paths based solely on an unverified package name.
- Do not weaken SELinux, linker namespace boundaries, or Android process isolation to make a test pass.

### Diagnostics and privacy

- Diagnostics must avoid recording secrets, private user content, or complete application memory dumps by default.
- Crash evidence should contain identifiers needed for reproduction, not unrestricted payload data.
- Any opt-in telemetry requires a separate design review and a documented retention policy.

## Security acceptance

Before advancing past a stage, tests must demonstrate rejection of at least one malformed input relevant to that stage and must verify that rejection is controlled, classified, and does not crash the Android runtime.

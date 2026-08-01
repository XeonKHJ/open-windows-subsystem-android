# Reliability Requirements

## Reproducibility

- The active plan must name exact commands for build, install, run, collect, and clean/reset.
- The environment baseline must record the AOSP revision, Android target, kernel/VM identity, host tool versions, and test APK version.
- R0 acceptance runs begin from a clean VM snapshot or equivalent reset state.

## Test ladder

1. Unit tests validate pure instruction, ELF, relocation, and ABI behavior.
2. Integration tests validate Native Bridge adapter and linker interactions.
3. APK end-to-end tests validate ART → bridge → native call flow.
4. Stress tests validate repeatability, leaks, thread behavior, and recovery.
5. Differential tests compare interpreter and JIT semantics when JIT exists.

## Diagnostics

Every native failure must include a stable `run_id`, build revision, process/package, library, requested ABI, error class, and references to relevant logs. ARM64 execution failures must additionally include PC, instruction bytes, decoded instruction if available, register state, and translator mode.

## Recovery behavior

- JIT failure falls back to interpreter only when doing so is semantically safe and explicitly logged.
- Unsupported behavior must stop the affected execution path with a controlled error; it must not continue with guessed semantics.
- A failing application must not destabilize zygote, ART, linker, or unrelated Android processes.

## Acceptance evidence retention

The active execution plan records commands and expected results. PRs attach or link to the evidence. Completed plans preserve the final evidence summary before moving under `docs/exec-plans/completed/`.

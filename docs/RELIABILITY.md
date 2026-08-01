# Reliability Requirements

## Reproducibility

- The active plan must name exact commands for build, install, run, collect, and clean/reset.
- The environment baseline must record the AOSP revision, Android target, kernel/VM identity, host tool versions, and test APK version.
- R0.0 acceptance requires two clean builds from the pinned manifest with matching image hashes.
- R0.1 acceptance runs begin from a clean VM snapshot or equivalent reset state.
- R0.2 acceptance runs begin from the R0.1 clean-boot reset state.
- R1 acceptance begins only after R0.0, R0.1, and R0.2 gates are all complete.

## Build baseline vs. product target

The AOSP x86_64 build target selected in R0.0 (e.g., `sdk_phone_x86_64-userdebug` or `aosp_x86_64-userdebug`) is a **development and userspace validation baseline**. It is not automatically a bootable Hyper-V product target. The Hyper-V-specific product target (tentatively `owsa_hyperv_x86_64-userdebug` or equivalent) is a separate decision to be validated and recorded in R0.1. See `docs/aosp-hyperv-bringup.md`.

## Test ladder

1. Unit tests validate pure instruction, ELF, relocation, and ABI behavior.
2. Integration tests validate Native Bridge adapter and linker interactions.
3. APK end-to-end tests validate ART → bridge → native call flow.
4. Stress tests validate repeatability, leaks, thread behavior, and recovery.
5. Differential tests compare interpreter and JIT semantics when JIT exists.

## Diagnostics

Every native failure must include a stable `run_id`, build revision, process/package, library, requested ABI, error class, and references to relevant logs. ARM64 execution failures must additionally include PC, instruction bytes, decoded instruction if available, register state, and translator mode.

Failure categories must be classified at the appropriate layer:

- **R0.1 boot failures**: `hyper-v`, `kernel`, `android-init`, `other`.
- **R0.2 / R1 run failures**: `adb`, `apk-install`, `art-jni`, `native-crash`, `boot`, `other`; plus lower-layer categories when applicable.

## Recovery behavior

- JIT failure falls back to interpreter only when doing so is semantically safe and explicitly logged.
- Unsupported behavior must stop the affected execution path with a controlled error; it must not continue with guessed semantics.
- A failing application must not destabilize zygote, ART, linker, or unrelated Android processes.

## Acceptance evidence retention

The active execution plan records commands and expected results. PRs attach or link to the evidence. Completed plans preserve the final evidence summary before moving under `docs/exec-plans/completed/`.

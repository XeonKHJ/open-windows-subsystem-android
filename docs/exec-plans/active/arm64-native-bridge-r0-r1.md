# Execution Plan: ARM64 Native Bridge R0–R1

- **Status:** Active
- **Owner:** Unassigned
- **Scope:** AOSP x86_64 build baseline, Hyper-V Android guest boot baseline, ADB/diagnostics/x86_64 JNI baseline, and AOSP Native Bridge adapter control-flow proof.
- **Out of scope:** ARM64 instruction execution, ELF relocation implementation, JIT, ARM32, app-window integration, notifications, and production compatibility claims.
- **Governing references:** `AGENTS.md`, `ARCHITECTURE.md`, `docs/design-docs/core-beliefs.md`, `docs/arm64-native-bridge-harness-plan.md`, `docs/aosp-hyperv-bringup.md`, `docs/QUALITY_SCORE.md`, `docs/RELIABILITY.md`, `docs/SECURITY.md`.

## Outcome

At the end of this plan, a controlled test APK has two independently verifiable paths:

1. An x86_64 JNI library loads and returns `nativeAdd(40, 2) = 42` directly on a self-built Android x86_64 image that reliably boots in Hyper-V.
2. An ARM64 JNI library is recognized as foreign ABI and its library-load/JNI-symbol request reaches `libowsa_nativebridge.so`; the adapter returns a structured `not_implemented` result without crashing ART, linker, zygote, or the process.

Successful AOSP compilation alone does **not** satisfy the R0 gate. The advance gate is a self-built x86_64 Android image that reliably boots in Hyper-V and passes direct x86_64 JNI validation. R1 cannot start until R0.0, R0.1, and R0.2 are all complete.

This proves the repository, environment, tests, diagnostics, and AOSP integration boundary before implementation starts on an ARM64 loader or translator.

---

## Harness loop

Each task follows this loop:

```text
Intent → inspect repository/environment → implement a narrow change
      → run local verification → inspect logs/metrics/evidence
      → self-review against invariants → update docs/plan
      → open PR → address review feedback → merge or revise
```

If an agent cannot verify a task because a command, fixture, log collector, or invariant is missing, it must add that enabling capability as a separate narrow task. It must not claim success from static inspection alone.

---

## Workstream A — R0: reproducible platform baseline

R0 is divided into three ordered sub-stages. Each sub-stage must pass its gate before the next begins.

```text
R0.0 — AOSP x86_64 build baseline
R0.1 — Hyper-V Android guest boot baseline      (requires R0.0)
R0.2 — ADB, diagnostics, and x86_64 JNI baseline (requires R0.1)
```

---

### R0.0 — AOSP x86_64 build baseline

**Intent**

Establish a reproducible, pinned AOSP x86_64 build that can be rebuilt identically by any developer or agent given the recorded environment. Completing this stage proves that the build infrastructure is under control before any boot or runtime claims are made.

**Preconditions / inputs**

- A supported 64-bit Linux AOSP build host (Ubuntu 20.04 LTS or Ubuntu 22.04 LTS recommended by AOSP; other distributions require explicit validation and recording).
- At least 400 GB free disk, 16 GB RAM, and sufficient swap for a full AOSP build.
- `repo` tool, the host toolchain package set named in the selected AOSP branch's `build/make/README.md`, and a fixed JDK version — all recorded with exact versions.
- Network access to AOSP Gerrit and any required dependency mirrors, or an offline mirror with a recorded manifest.

**Implementation steps**

1. Select and pin a non-moving AOSP revision: prefer a named release tag (e.g., `android-14.0.0_r21`) or a manifest revision that will not be rebased; document the exact `repo init -u ... -b <branch> --manifest-name <manifest> --repo-rev <rev>` invocation used.
2. Record the selected product target and build variant. Use a standard x86_64 `userdebug` target (e.g., `sdk_phone_x86_64-userdebug` or `aosp_x86_64-userdebug`) as the build/userspace validation baseline. This target is a development control baseline only; it is **not** automatically a bootable Hyper-V product target — see `docs/aosp-hyperv-bringup.md` for the distinction.
3. Add `docs/environment-baseline.md` recording: AOSP manifest URL, branch/tag/manifest revision, product target, build variant, kernel revision (from the pinned source), host OS name and version, host tool versions (make, Python, Java, clang, repo), build commands, image hashes (SHA-256 of each output image), and build log location.
4. Define an environment fingerprint format (e.g., a JSON object) that test output references.
5. Perform the build and record the build log.
6. Perform a second clean build (delete `out/` and rebuild) to confirm the build is reproducible and to detect environment-dependent artifacts.

**Required repository outputs**

- `docs/environment-baseline.md` with all fields above populated.
- A committed environment fingerprint schema example.
- Build log samples or a reference to their storage location.
- Recorded SHA-256 hashes of all output images from both clean builds.

**Acceptance and verification steps**

1. Run two independent clean builds from the pinned manifest.
2. Compare SHA-256 hashes of all output images between the two builds; identical hashes confirm a reproducible build.
3. If hashes differ, identify and document every source of non-determinism; resolve or explicitly accept each one before advancing.
4. Confirm `docs/environment-baseline.md` is complete enough for a second developer or agent to reproduce the exact build without asking additional questions.
5. Confirm the document explicitly states that the selected x86_64 build target is a development baseline and is not the eventual Hyper-V product target.

**Evidence to retain**

- `artifacts/r0.0/<run_id>/environment.json` — environment fingerprint.
- `artifacts/r0.0/<run_id>/image-hashes.txt` — SHA-256 of each output image from both builds.
- `artifacts/r0.0/<run_id>/build-log-summary.txt` — build timing, warnings, and final image list.
- A link or inline excerpt in the PR description.

**Gate criteria**

- Two clean builds complete without error.
- Image hashes are identical across both builds, or every hash difference is explained and accepted.
- `docs/environment-baseline.md` is present, complete, and reviewed.
- The document clearly distinguishes the build-validation target from an eventual Hyper-V product target.

**Stop / rollback rule**

Do not proceed to R0.1 while the build is non-reproducible or the environment baseline is incomplete. Fix environment drift and rerun both builds before advancing.

**Main risks**

- AOSP build dependencies change between invocations (network-fetched tools, Java version drift).
- The pinned tag may require a host toolchain version not natively available; document any workaround.
- Build non-determinism from timestamps embedded in images.

---

### R0.1 — Hyper-V Android guest boot baseline

**Intent**

Define the Hyper-V boot-chain and image-delivery design, then boot the AOSP x86_64 image built in R0.0 as a minimal Android guest in Hyper-V and confirm reproducible boot completion. This stage must define the design before claiming success — do not claim "boot works" without first recording the kernel, initramfs, partition layout, virtual disk approach, and Hyper-V device configuration used.

**Preconditions / inputs**

- R0.0 gate passed.
- A Windows host with Hyper-V enabled (Generation 2 VM or Generation 1 depending on kernel/bootloader requirements; the choice must be recorded and justified).
- The AOSP x86_64 images built and hashed in R0.0.
- Access to Hyper-V PowerShell or Hyper-V Manager for VM creation, console attachment, and log collection.

**Implementation steps**

1. Define and document the Hyper-V boot-chain design before any boot attempt:
   - Kernel image and command line.
   - Initramfs usage (present or absent; justified).
   - Android partition/image layout (system, vendor, data, ramdisk — which images are used and how they are delivered to the guest).
   - Virtual disk approach (VHD/VHDX, raw, or direct image presentation; document and justify the choice).
   - Console/serial logging configuration (how early kernel messages are captured from the host).
   - Storage and network virtual devices (which Hyper-V device model is used; record model identifiers).
   - Memory and CPU configuration (vCPUs, RAM, NUMA topology; record all non-default settings).
2. Record all of the above in `docs/environment-baseline.md` under a dedicated `Hyper-V guest configuration` section, and capture them in the environment fingerprint.
3. Perform an initial boot attempt. Collect: kernel early boot log (from console), Android init log, logcat when Android reaches that stage, and relevant host-side Hyper-V event log entries.
4. Verify `sys.boot_completed=1` via either a host-side ADB bridge or a serial/console output marker.
5. Establish a clean-boot reset procedure: either a Hyper-V snapshot restored to pre-boot state, or an equivalent deterministic reset mechanism. Document the reset command or sequence.
6. Run ten clean-start boot attempts from the reset state. For each attempt, record: attempt number, start time, time to `sys.boot_completed` or failure event, and outcome classification.
7. Classify any failures into one of: `hyper-v` (VM creation, device, or hypervisor error), `kernel` (kernel panic, early init failure), `android-init` (Android init or services failure), or `other` (with a description).

**Required repository outputs**

- `docs/environment-baseline.md` updated with Hyper-V guest configuration.
- `docs/hyperv-boot-design.md` — a short document recording the boot-chain and image-delivery decisions, the rationale for each choice, and any open questions.
- A reset procedure script or documented command sequence.
- Boot-attempt log samples in the evidence directory.

**Acceptance and verification steps**

1. Inspect `docs/hyperv-boot-design.md` and confirm it is present and complete before evaluating any boot results.
2. Confirm the reset procedure is documented and can be executed without manual steps beyond those listed.
3. Review the ten-run log summary and confirm at least nine of ten attempts reached `sys.boot_completed=1`.
4. Confirm every failed attempt is classified into one of the defined categories.
5. Confirm the evidence directory contains kernel boot log, Android init log, and host-side Hyper-V diagnostics for at least one representative successful run.

**Evidence to retain**

- `artifacts/r0.1/<run_id>/boot-attempts.csv` — one row per attempt: attempt index, outcome, time-to-boot-completed or failure event, failure category if applicable.
- `artifacts/r0.1/<run_id>/kernel-boot.log` — kernel dmesg from one representative successful run.
- `artifacts/r0.1/<run_id>/android-init.log` — Android init output from the same run.
- `artifacts/r0.1/<run_id>/logcat-boot.log` — logcat from guest boot when available.
- `artifacts/r0.1/<run_id>/hyperv-events.txt` — relevant host-side Hyper-V event log entries.

**Gate criteria**

- `docs/hyperv-boot-design.md` is present and reviewed before advancing.
- At least 9 of 10 clean-start attempts reach `sys.boot_completed=1`.
- Every failed attempt is classified.
- Evidence files are present and inspectable.

**Stop / rollback rule**

Do not proceed to R0.2 while boot success rate is below 9/10 or the boot-chain design is undocumented. Fix boot reliability, update the design document, and re-run the ten-attempt sequence before advancing.

**Main risks**

- The standard AOSP x86_64 build target may not boot directly under Hyper-V without kernel configuration changes or a custom bootloader; this is expected and must be investigated and documented rather than assumed to work.
- Hyper-V Generation 2 UEFI and Secure Boot may require additional configuration.
- Console/serial log capture may require specific Hyper-V settings not enabled by default.
- The eventual `owsa_hyperv_x86_64-userdebug` product target (see `docs/aosp-hyperv-bringup.md`) will differ from the standard build-baseline target; document gaps as they are discovered.

---

### R0.2 — ADB, diagnostics, and x86_64 JNI baseline

**Intent**

Establish stable ADB connectivity to the running Hyper-V guest, implement a one-command build/install/run/collect/reset workflow, validate that a pure Java/Kotlin APK runs correctly, and confirm that a direct x86_64 JNI function call returns the expected deterministic result. This stage proves that the full test and evidence pipeline is operational before any Native Bridge adapter work begins.

**Preconditions / inputs**

- R0.1 gate passed.
- ADB host binary available on the build/test host; version recorded.
- `nativebridge/tests/apk/nativebridge-smoke/` source tree (to be created in this stage if not already present from earlier work).

**Implementation steps**

1. Establish ADB connectivity: configure the Hyper-V network or serial-over-USB such that `adb devices` shows the guest after boot. Document the exact ADB connection method (TCP/IP forwarding, USB pass-through, or other).
2. Verify ADB stability: confirm that the ADB connection survives a full boot cycle without manual intervention.
3. Create `nativebridge/tests/apk/nativebridge-smoke/` if not present:
   - Implement one Java/Kotlin call surface: `nativeAdd(int, int): int`.
   - Build an x86_64 `libdemo.so` that returns `42` for input `(40, 2)`.
   - Build an ARM64 `libdemo.so` with the same exported JNI surface and deterministic implementation (used later in R1; do not install in R0.2).
   - Ensure the x86_64-only APK variant packages only `lib/x86_64/libdemo.so`.
4. Implement and document a one-command workflow:
   - `build` — compile the smoke APK.
   - `install` — install it on the running guest via ADB.
   - `run` — invoke the test and capture test output.
   - `collect` — gather logcat, tombstones if present, host-side Hyper-V logs, test result, environment fingerprint, and timestamps into a single `artifacts/r0.2/<run_id>/` directory.
   - `reset` — restore guest to clean-boot state via the R0.1 reset procedure.
5. Install and run a pure Java/Kotlin APK (no native library) to confirm the ART runtime is functional independent of any JNI path.
6. Install the x86_64 smoke APK and verify `nativeAdd(40, 2) = 42` and that the log records the library path and ABI as `x86_64`.
7. Run ten clean-baseline attempts from the reset state. For each attempt, record: outcome, return value if applicable, ABI reported in logs, and failure category if applicable.
8. Classify any failures as: `adb` (connection or command failure), `apk-install` (package manager rejection), `art-jni` (ART or JNI runtime error), `native-crash` (tombstone or SIGSEGV), `boot` (guest failed to reach ready state), or `other` (with description).
9. Document the `collect` artifact layout in `docs/test-protocol.md`.

**Required repository outputs**

- `nativebridge/tests/apk/nativebridge-smoke/` with x86_64 and ARM64 library variants.
- One-command workflow scripts (or documented command sequences) for build, install, run, collect, and reset.
- `docs/test-protocol.md` describing invocation and expected artifact layout.
- Ten-run evidence artifacts in `artifacts/r0.2/`.

**Acceptance and verification steps**

1. Execute the one-command workflow end-to-end for one successful run and confirm the artifact directory is produced with all required files.
2. Execute one controlled failing run (e.g., install the ARM64-only APK without Native Bridge to confirm a classified failure), run `collect`, and confirm the artifact shows a clear classified failure.
3. Review the ten-run summary and confirm at least 9 of 10 attempts return `nativeAdd(40, 2) = 42` with `x86_64` ABI in logs.
4. Confirm every failed attempt is classified.
5. Confirm the pure Java/Kotlin APK ran successfully on at least one attempt.

**Evidence to retain**

- `artifacts/r0.2/<run_id>/test-result.json` — structured result: outcome, return value, ABI, run_id, environment fingerprint reference.
- `artifacts/r0.2/<run_id>/logcat.log` — full logcat from the run.
- `artifacts/r0.2/<run_id>/tombstones/` — any tombstone files generated.
- `artifacts/r0.2/<run_id>/hyperv-events.txt` — host-side diagnostic log.
- `artifacts/r0.2/<run_id>/environment.json` — environment fingerprint snapshot.
- A ten-run summary CSV: attempt index, outcome, return value, ABI, failure category.

**Gate criteria**

- ADB connects reliably after every clean-boot reset without manual steps.
- The one-command workflow is documented and produces a complete artifact directory.
- At least 9 of 10 clean-baseline attempts return `nativeAdd(40, 2) = 42` with `x86_64` ABI.
- Every failed attempt is classified into the defined categories.
- The pure Java/Kotlin APK ran successfully.
- `docs/test-protocol.md` is present and reviewed.

**Stop / rollback rule**

Do not begin R1 (Native Bridge adapter work) until R0.2 gate passes. If x86_64 direct JNI is not stable, fix the boot, ADB, or ART issue and re-run the ten-attempt sequence.

**Main risks**

- ADB connectivity over Hyper-V network may require specific VM switch configuration.
- ABI selection in the APK may be ambiguous if multiple ABI libraries are bundled; the x86_64-only APK variant must be verified.
- ART version mismatches between the AOSP build and the test APK compile SDK may cause install failures.

---

## Workstream B — R1: AOSP Native Bridge adapter proof

### B1. Freeze the AOSP integration contract

**Intent**

Prevent implementation against guessed or stale Native Bridge APIs.

**Implementation steps**

1. Inspect the exact AOSP revision selected in R0.0.
2. Record the Native Bridge interface version, callback names, initialization order, relevant system properties, linker namespace behavior, and ABI-routing observations in `nativebridge/docs/nativebridge-integration.md`.
3. Link the source paths and exact revision used for the investigation.
4. List open unknowns explicitly; do not hide them in prose.

**Acceptance steps**

1. Review the contract against the selected source tree.
2. Confirm every adapter callback that will be implemented is mapped to a source location and expected behavior.
3. Confirm no source from a different AOSP revision is treated as authoritative.

**Evidence**

- Source-path table in the integration document.
- Review checklist with zero unresolved blocker-level unknowns.

**Stop rule**

If the interface cannot be identified or the foreign-ABI route cannot be triggered, pause and create an investigation task; do not build a speculative loader.

### B2. Implement the adapter stub

**Intent**

Prove that AOSP routes the ARM64 load request into OWSA code.

**Implementation steps**

1. Add `nativebridge/Android.bp` and build `libowsa_nativebridge.so` for the target Android x86_64 system image.
2. Implement only the adapter callbacks required to initialize, recognize supported foreign ABI/path input, receive the library-load request, and receive JNI trampoline/symbol queries where applicable.
3. Parse enough of the candidate file to identify ELF64/AArch64 safely.
4. Emit structured events: `bridge_initialized`, `foreign_elf_recognized`, `library_load_requested`, `jni_symbol_requested`, and `not_implemented`.
5. Return a controlled, documented `not_implemented` result for code-execution paths.

**Acceptance steps**

1. Boot the clean guest and verify `bridge_initialized`.
2. Run the direct x86_64 fixture and confirm it still returns `42` without entering the foreign path.
3. Install/run the ARM64-only fixture.
4. Verify the event sequence reaches `foreign_elf_recognized` and the applicable load/symbol request event.
5. Verify the application fails in the expected controlled way, with a nonzero classified result, not a process/zygote/runtime crash.
6. Repeat the ARM64 test 100 times; require zero ART, linker, and zygote crashes.

**Evidence**

- Structured event sequence per run.
- Summary of 100 runs.
- Any tombstones and their classification; the target is none caused by adapter handling.

**Stop rule**

Do not start ELF mapping, relocation, instruction decoding, or JIT until this gate passes.

### B3. Make R1 mechanically reviewable

**Intent**

Turn R1 principles into checks rather than relying on reviewer memory.

**Implementation steps**

1. Add an integration test that fails if the ARM64 fixture does not reach the expected adapter event.
2. Add a test that fails if the x86_64 fixture enters the foreign adapter path.
3. Add a static/repository check that active plan and architecture documents exist and are linked from `AGENTS.md`.
4. Add a structured-diagnostics schema test for mandatory R1 fields.

**Acceptance steps**

1. Run all R0/R1 tests from the documented command.
2. Intentionally remove or alter one expected event in a local branch and verify the integration test fails.
3. Intentionally feed a malformed/non-AArch64 candidate and verify the adapter rejects it with a classified error rather than crashing.

**Evidence**

- Passing test report.
- Demonstration of each negative test failing for the intended reason.

---

## Final R0/R1 acceptance gate

**Successful AOSP compilation alone does not satisfy this gate.** The advance criterion is a self-built x86_64 Android image that reliably boots in Hyper-V and passes direct x86_64 JNI validation. R1 cannot start until R0.0, R0.1, and R0.2 are all complete.

Mark this plan **Completed** only when all statements are true:

**R0.0 — AOSP x86_64 build baseline**
- [ ] Two clean builds complete without error from the pinned AOSP manifest.
- [ ] Image hashes are identical across both builds, or every hash difference is explained and accepted.
- [ ] `docs/environment-baseline.md` is present, complete, and reviewed.
- [ ] The document explicitly distinguishes the build-validation target from an eventual Hyper-V product target.

**R0.1 — Hyper-V Android guest boot baseline**
- [ ] `docs/hyperv-boot-design.md` is present and reviewed before any boot results are claimed.
- [ ] At least 9 of 10 clean-start boot attempts reach `sys.boot_completed=1`.
- [ ] Every failed boot attempt is classified as `hyper-v`, `kernel`, `android-init`, or `other`.
- [ ] Boot evidence (kernel log, init log, logcat, host Hyper-V events) is retained.

**R0.2 — ADB, diagnostics, and x86_64 JNI baseline**
- [ ] ADB connects reliably after every clean-boot reset without manual steps.
- [ ] The one-command workflow (build/install/run/collect/reset) is documented and produces a complete artifact directory.
- [ ] A pure Java/Kotlin APK runs successfully on the guest.
- [ ] The x86_64 JNI smoke APK returns `nativeAdd(40, 2) = 42` in at least 9 of 10 clean-baseline attempts.
- [ ] Every failed attempt is classified as `adb`, `apk-install`, `art-jni`, `native-crash`, `boot`, or `other`.
- [ ] `docs/test-protocol.md` is present and reviewed.

**R1 — Native Bridge adapter proof**
- [ ] The AOSP Native Bridge integration contract is tied to the exact selected AOSP revision.
- [ ] `libowsa_nativebridge.so` initializes in the Android guest.
- [ ] The ARM64 fixture reaches the adapter's foreign-ELF and applicable load/symbol-request events.
- [ ] The x86_64 fixture does not enter the foreign path.
- [ ] 100 ARM64 adapter runs cause no ART, linker, or zygote crash.
- [ ] Malformed/unsupported foreign input is rejected in a controlled and diagnosed manner.
- [ ] Quality, reliability, and security documents are still accurate.

## Completion and handoff

When the final gate passes:

1. Add a final evidence summary under this plan.
2. Move this file to `docs/exec-plans/completed/` without rewriting history.
3. Create a new active R2 plan for ELF mapping, relocation scope, interpreter instruction subset, and Java→ARM64 JNI primitive trampoline.
4. Do not begin JIT planning in code until the R2 interpreter gate is passed.

## Decision log

| Date | Decision | Rationale | Consequence |
|---|---|---|---|
| 2026-08-01 | Scope current work to R0/R1 | Adapter control-flow proof is the prerequisite for executable translation work | No ELF execution or JIT work in this plan |
| 2026-08-01 | Use repository-local plans and evidence | Agents need durable, versioned context | PRs must update plans/docs with implementation changes |
| 2026-08-01 | Split R0 into R0.0/R0.1/R0.2 | AOSP build, Hyper-V boot, and x86_64 JNI baseline are each independently verifiable and must not be collapsed | R1 is blocked until all three sub-stages pass; build success alone is not sufficient |
| 2026-08-01 | Standard x86_64 build target is a development baseline only | A stock Cuttlefish-style target may not boot directly under Hyper-V; the eventual product target is `owsa_hyperv_x86_64-userdebug` or equivalent | R0.0 records the build baseline; R0.1 documents Hyper-V-specific boot-chain design before claiming boot success |

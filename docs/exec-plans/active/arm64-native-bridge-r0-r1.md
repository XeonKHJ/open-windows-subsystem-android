# Execution Plan: ARM64 Native Bridge R0–R1

- **Status:** Active
- **Owner:** Unassigned
- **Scope:** Reproducible experiment baseline and AOSP Native Bridge adapter control-flow proof.
- **Out of scope:** ARM64 instruction execution, ELF relocation implementation, JIT, ARM32, app-window integration, notifications, and production compatibility claims.
- **Governing references:** `AGENTS.md`, `ARCHITECTURE.md`, `docs/design-docs/core-beliefs.md`, `docs/arm64-native-bridge-harness-plan.md`, `docs/QUALITY_SCORE.md`, `docs/RELIABILITY.md`, `docs/SECURITY.md`.

## Outcome

At the end of this plan, a controlled test APK has two independently verifiable paths:

1. An x86_64 JNI library loads and returns `nativeAdd(40, 2) = 42` directly.
2. An ARM64 JNI library is recognized as foreign ABI and its library-load/JNI-symbol request reaches `libowsa_nativebridge.so`; the adapter returns a structured `not_implemented` result without crashing ART, linker, zygote, or the process.

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

### A1. Establish the environment manifest

**Intent**

Make every R0 run attributable to a specific Android and VM environment.

**Implementation steps**

1. Add `docs/environment-baseline.md`.
2. Record AOSP source revision, manifest revision, Android product/build target, kernel revision, Hyper-V host/guest configuration identifier, disk-image provenance, and host tool versions.
3. Define an environment fingerprint format used by test output.
4. Add a reset procedure that starts from a clean VM snapshot or an equivalent known state.

**Acceptance steps**

1. Run the reset procedure twice.
2. Collect the environment fingerprint after each reset.
3. Confirm the fingerprints are either identical or explain every permitted difference.
4. Confirm the document contains enough information for a second developer/agent to recreate the test target.

**Evidence**

- `artifacts/r0/<run_id>/environment.json` or an equivalent committed/test artifact.
- A link or excerpt in the PR description.

**Stop rule**

Do not begin adapter integration while the environment cannot be reset and identified reproducibly.

### A2. Create the ABI-controlled smoke APK

**Intent**

Have one application-level fixture that isolates ABI routing from application complexity.

**Implementation steps**

1. Create `nativebridge/tests/apk/nativebridge-smoke/`.
2. Implement one Java/Kotlin call surface: `nativeAdd(int, int): int`.
3. Build an x86_64 `libdemo.so` whose result is `42` for input `(40, 2)`.
4. Build an ARM64 `libdemo.so` with the same exported JNI surface and deterministic implementation.
5. Ensure each test mode packages only the intended ABI, or document and test the ABI-selection behavior explicitly.

**Acceptance steps**

1. Install the x86_64 fixture.
2. Run the test activity/instrumentation command.
3. Verify return value `42` and library path/ABI reported by logs.
4. Run from the clean baseline ten times.
5. Require at least 9 successful runs; classify every failure as boot, ADB, package install, ART/JNI, or native crash.

**Evidence**

- Test output showing `nativeAdd(40, 2) = 42`.
- ABI/library-path log record for every run.
- Failure classification table when any attempt fails.

**Stop rule**

No ARM64 adapter work until x86_64 direct JNI is stable.

### A3. Add one-command collection

**Intent**

Let agents verify a run without manually assembling logs.

**Implementation steps**

1. Add scripts for `build`, `install`, `run`, `collect`, and `reset`.
2. Make `collect` gather logcat, tombstones if present, host-side VM logs needed for diagnosis, test result, environment fingerprint, and timestamps.
3. Produce a single run directory keyed by `run_id`.
4. Document invocation and expected artifact layout in `docs/test-protocol.md`.

**Acceptance steps**

1. Execute one successful x86_64 run.
2. Execute one controlled failing run, such as an intentionally invalid package/library selection.
3. Run `collect` for both.
4. Confirm both artifact sets can be inspected without rerunning the test and contain a clear success/failure classification.

**Evidence**

- Two sample artifact manifests.
- Command transcript and exit status.

**Stop rule**

Do not accept “works on my machine” output or screenshots as R0 evidence.

---

## Workstream B — R1: AOSP Native Bridge adapter proof

### B1. Freeze the AOSP integration contract

**Intent**

Prevent implementation against guessed or stale Native Bridge APIs.

**Implementation steps**

1. Inspect the exact AOSP revision selected in A1.
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

Mark this plan **Completed** only when all statements are true:

- [ ] The environment manifest and reset process are documented and reproducible.
- [ ] The x86_64 direct-JNI fixture returns `42` in at least 9 of 10 clean-baseline attempts.
- [ ] One-command run collection produces inspectable, classified evidence.
- [ ] The AOSP Native Bridge integration contract is tied to the exact selected AOSP revision.
- [ ] `libowsa_nativebridge.so` initializes in the Android guest.
- [ ] The ARM64 fixture reaches the adapter’s foreign-ELF and applicable load/symbol-request events.
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

# AOSP / Hyper-V Bring-Up Procedure and Evidence Checklist

> **Scope:** R0 operator and agent guide. This document gives practical procedure steps and an evidence checklist for R0.0, R0.1, and R0.2. It does not contain verified build scripts or claim that any stock AOSP target boots directly under Hyper-V. All product-target and boot-chain decisions are marked as items to be validated and captured in the environment baseline.
>
> **Governing plan:** `docs/exec-plans/active/arm64-native-bridge-r0-r1.md`

---

## Target distinction: build baseline vs. Hyper-V product target

Two distinct targets appear in the R0 work. They must not be confused:

| Target | Purpose | Status |
|---|---|---|
| **AOSP x86_64 build/userspace validation target** (e.g., `sdk_phone_x86_64-userdebug` or `aosp_x86_64-userdebug`) | Confirms the AOSP source tree builds cleanly, ART and userspace libraries are present, and images are reproducible. Used as the R0.0 development control baseline. | Available in AOSP; must be selected and pinned in R0.0. |
| **`owsa_hyperv_x86_64-userdebug`** (or an equivalently named custom product target) | The eventual OWSA-specific product target tuned for Hyper-V boot: correct kernel configuration, Hyper-V virtual device drivers, appropriate partitioning, console/serial settings. | Does **not** exist yet; must be designed, created, and validated starting in R0.1. |

**Why the former does not prove the latter:**

- The standard Cuttlefish-oriented x86_64 AOSP target is designed for the Cuttlefish virtual device, which uses specific virtual device drivers (virtio, crosvm) and a purpose-built launcher. It is not configured for Hyper-V's device model.
- Booting under Hyper-V requires a kernel configured for Hyper-V's synthetic devices (SCSI, network, framebuffer) and potentially a different bootloader path (UEFI/GRUB vs. the Cuttlefish boot flow).
- Successfully compiling `sdk_phone_x86_64-userdebug` proves the build environment is functional. It does not prove that the resulting images will boot in Hyper-V.
- R0.1 must document the specific boot-chain design choices before claiming boot success. These choices must be recorded as architecture decisions and retained as environment baseline entries.

---

## R0.0 — AOSP x86_64 Build Baseline: Procedure Checklist

### Environment setup checklist

- [ ] Host OS is a supported 64-bit Linux distribution for AOSP builds; version recorded in `docs/environment-baseline.md`.
- [ ] All AOSP build dependencies are installed; package list and versions recorded.
- [ ] `repo` tool is installed; version recorded.
- [ ] JDK version is pinned to what the selected AOSP branch requires; version recorded.
- [ ] Available disk is ≥ 400 GB for source, build artifacts, and outputs.
- [ ] Available RAM is ≥ 16 GB; swap configured if RAM is borderline.

### Source selection and pinning checklist

- [ ] A specific AOSP release tag or manifest revision is selected (e.g., `android-14.0.0_r21`). Floating branch names that may be rebased are not acceptable.
- [ ] The exact `repo init` command is recorded in `docs/environment-baseline.md`, including `-u`, `-b`, `--manifest-name`, and `--repo-rev` (or equivalent pinning flags).
- [ ] The manifest is synced and the sync command (with `-j` thread count) is recorded.
- [ ] The selected product target and build variant are recorded. Example: `sdk_phone_x86_64-userdebug`. This is the **build/userspace validation baseline**.
- [ ] `docs/environment-baseline.md` explicitly states: *"This product target is used as a build and userspace validation baseline. It is not the Hyper-V product target and its images are not assumed to boot directly under Hyper-V."*

### Build procedure checklist

- [ ] `source build/envsetup.sh` and `lunch <target>` are run; the exact invocation is recorded.
- [ ] Build is started: `m` (or equivalent); build command with parallelism flags is recorded.
- [ ] Build completes without errors. Build warnings are noted if unusual.
- [ ] Build log is saved to `artifacts/r0.0/<run_id>/build-log.txt` or an equivalent persistent location.
- [ ] Output images are listed; SHA-256 of each image is computed and recorded in `artifacts/r0.0/<run_id>/image-hashes.txt`.

### Reproducibility verification checklist (two-build check)

- [ ] `out/` directory is deleted or a fresh build host state is established.
- [ ] Second clean build is run with the same `lunch` target and build command.
- [ ] SHA-256 hashes of all output images from the second build are computed.
- [ ] Hashes are compared against the first build. Result: identical or each difference documented.
- [ ] If hashes differ, each source of non-determinism is identified (e.g., embedded build timestamps) and either resolved or explicitly accepted with rationale.

### R0.0 evidence to retain

```
artifacts/r0.0/<run_id>/
  environment.json         # environment fingerprint (AOSP rev, target, host tools, etc.)
  image-hashes.txt         # SHA-256 of each output image, both builds
  build-log.txt            # build log or reference to storage location
  build-log-build2.txt     # second clean build log
docs/environment-baseline.md  # all fields populated
```

### R0.0 gate (self-check before proceeding to R0.1)

- [ ] Both clean builds complete without error.
- [ ] Image hashes are identical, or every difference is documented and accepted.
- [ ] `docs/environment-baseline.md` is complete and reviewed.
- [ ] The document clearly states the build target is a development baseline, not the Hyper-V product target.

---

## R0.1 — Hyper-V Android Guest Boot Baseline: Procedure Checklist

> **Rule:** Do not claim boot success before documenting the boot-chain design. Complete the design checklist below first.

### Boot-chain design checklist (must be complete before first boot attempt)

Record all decisions in `docs/hyperv-boot-design.md`. Mark each item as a **decision made and rationale recorded** or **open question to be resolved**:

- [ ] Hyper-V VM generation (Generation 1 or Generation 2): decision made, rationale recorded.
- [ ] Kernel image path and command line: recorded (or noted as to-be-determined with investigation plan).
- [ ] Initramfs: present or absent; decision made and justified.
- [ ] Android partition/image layout: which images (system, vendor, data, ramdisk) are used and how they are delivered (VHDX, raw image, direct attach); decision made and justified.
- [ ] Virtual disk approach: VHD/VHDX or other; format and provisioning type recorded.
- [ ] Console/serial logging configuration: how early kernel messages are captured from the Hyper-V host; method recorded.
- [ ] Storage virtual devices: Hyper-V device model used (e.g., Hyper-V SCSI controller); model identifiers recorded.
- [ ] Network virtual devices: Hyper-V network adapter type; configuration recorded.
- [ ] Memory allocation: vRAM in MB; recorded.
- [ ] vCPU count: recorded.
- [ ] Any Hyper-V-specific kernel configuration requirements identified during investigation: listed as resolved or open.

### Boot attempt procedure checklist

- [ ] VM is created according to the documented design.
- [ ] Initial boot attempt is run; kernel early boot log, Android init log, and logcat (when available) are collected.
- [ ] `sys.boot_completed=1` is verified by ADB, serial console marker, or logcat grep; the verification method is documented.
- [ ] Clean-boot reset procedure is documented (Hyper-V snapshot restore command or equivalent).
- [ ] Ten clean-start attempts are run from the reset state.
- [ ] For each attempt: attempt index, start time, time to `sys.boot_completed` or failure event, outcome classification, and failure category (if failed) are recorded in `artifacts/r0.1/<run_id>/boot-attempts.csv`.

### Failure classification

Every failed boot attempt must be classified as one of:

| Category | Meaning |
|---|---|
| `hyper-v` | VM creation, device attachment, or hypervisor error before kernel starts |
| `kernel` | Kernel panic, early init failure, or kernel not starting Android init |
| `android-init` | Android init or early service failure after the kernel is running |
| `other` | Does not fit the above; must include a description |

### R0.1 evidence to retain

```
artifacts/r0.1/<run_id>/
  boot-attempts.csv         # one row per attempt: index, outcome, time-to-boot, failure category
  kernel-boot.log           # kernel dmesg from one representative successful run
  android-init.log          # Android init output from the same run
  logcat-boot.log           # logcat from guest boot (when available)
  hyperv-events.txt         # relevant host-side Hyper-V event log entries
docs/hyperv-boot-design.md  # all boot-chain design fields populated
```

### R0.1 gate (self-check before proceeding to R0.2)

- [ ] `docs/hyperv-boot-design.md` is present and reviewed.
- [ ] At least 9 of 10 clean-start attempts reached `sys.boot_completed=1`.
- [ ] Every failed attempt is classified.
- [ ] Evidence files are present and inspectable.

---

## R0.2 — ADB, Diagnostics, and x86_64 JNI Baseline: Procedure Checklist

### ADB connectivity checklist

- [ ] ADB connection method is defined and documented (TCP/IP forwarding, USB pass-through, or other).
- [ ] `adb devices` shows the guest after a clean boot without manual steps.
- [ ] ADB connection survives a full clean-boot cycle.
- [ ] ADB host binary version is recorded in `docs/environment-baseline.md`.

### Smoke APK checklist

- [ ] `nativebridge/tests/apk/nativebridge-smoke/` source tree is present.
- [ ] Java/Kotlin call surface `nativeAdd(int, int): int` is implemented.
- [ ] x86_64-only APK variant packages only `lib/x86_64/libdemo.so` (no ARM64 library in this variant).
- [ ] x86_64 `libdemo.so` returns `42` for input `(40, 2)` — verified by unit test or controlled run.
- [ ] ARM64 `libdemo.so` is built and available but is not installed in R0.2 runs (reserved for R1).

### One-command workflow checklist

- [ ] `build` command is documented and produces the smoke APK.
- [ ] `install` command installs the APK on the running guest via ADB.
- [ ] `run` command invokes the test and captures output.
- [ ] `collect` command gathers logcat, tombstones, Hyper-V host logs, test result JSON, environment fingerprint, and timestamps into `artifacts/r0.2/<run_id>/`.
- [ ] `reset` command restores guest to clean-boot state and waits for ADB reconnection.
- [ ] `docs/test-protocol.md` documents the invocation and expected artifact layout.

### Validation runs checklist

- [ ] Pure Java/Kotlin APK (no native library) is installed and runs successfully on the guest.
- [ ] x86_64 smoke APK is installed and `nativeAdd(40, 2)` returns `42`; log records ABI as `x86_64`.
- [ ] Ten clean-baseline attempts are run from the reset state.
- [ ] For each attempt: outcome, return value if applicable, ABI reported in logs, failure category if failed, are recorded.
- [ ] At least 9 of 10 attempts return `42` with `x86_64` ABI.
- [ ] Every failed attempt is classified.

### Failure classification

Every failed R0.2 run must be classified as one of:

| Category | Meaning |
|---|---|
| `adb` | ADB connection or command failure |
| `apk-install` | Package manager rejection or install error |
| `art-jni` | ART runtime or JNI error |
| `native-crash` | Tombstone or SIGSEGV in native code |
| `boot` | Guest failed to reach ready state before the test |
| `other` | Does not fit the above; must include a description |

### R0.2 evidence to retain

```
artifacts/r0.2/<run_id>/
  test-result.json         # outcome, return value, ABI, run_id, env fingerprint ref
  logcat.log               # full logcat from the run
  tombstones/              # any tombstone files generated
  hyperv-events.txt        # host-side Hyper-V diagnostic log
  environment.json         # environment fingerprint snapshot
  run-summary.csv          # ten-run summary: index, outcome, return value, ABI, failure category
docs/test-protocol.md      # invocation and artifact layout documented
```

### R0.2 gate (self-check before proceeding to R1)

- [ ] ADB connects reliably after every clean-boot reset without manual steps.
- [ ] One-command workflow is documented and produces a complete artifact directory.
- [ ] Pure Java/Kotlin APK ran successfully.
- [ ] At least 9 of 10 attempts return `nativeAdd(40, 2) = 42` with `x86_64` ABI.
- [ ] Every failed attempt is classified.
- [ ] `docs/test-protocol.md` is present and reviewed.

---

## Proceeding to R1

R1 (Native Bridge Adapter Stub) may not start until **all three** R0 sub-stage gates above are checked. The advance gate is:

> A self-built x86_64 Android image reliably boots in Hyper-V, passes direct x86_64 JNI validation, and all R0 evidence is committed to the repository.

Successful AOSP compilation alone does not satisfy this gate. See the final acceptance gate in `docs/exec-plans/active/arm64-native-bridge-r0-r1.md`.

---

## Cross-references

| Document | Purpose |
|---|---|
| `docs/exec-plans/active/arm64-native-bridge-r0-r1.md` | Authoritative executable acceptance steps and evidence requirements |
| `docs/environment-baseline.md` | Environment fingerprint (created during R0.0) |
| `docs/hyperv-boot-design.md` | Boot-chain and image-delivery design (created during R0.1) |
| `docs/test-protocol.md` | One-command workflow invocation and artifact layout (created during R0.2) |
| `ARCHITECTURE.md` | System component map and current stage |
| `docs/arm64-native-bridge-harness-plan.md` | Long-range R0–R5 roadmap |
| `docs/QUALITY_SCORE.md` | Quality gates and scorecard |
| `docs/RELIABILITY.md` | Reproducibility and test requirements |

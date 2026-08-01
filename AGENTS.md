# OWSA Agent Map

> This file is intentionally short. It is the navigation entry point for coding agents, not a complete manual.

## Mission

Build an Android runtime for Windows with an x86_64 Android guest on Hyper-V. The current research priority is an ARM64 Native Bridge that executes `arm64-v8a` native libraries on x86_64 without translating DEX/Java/Kotlin.

## Non-negotiable invariants

1. **DEX is not ARM code.** Java/Kotlin/DEX runs through x86_64 ART; only ARM64 ELF native code is translated.
2. **Scope is ARM64 first.** Do not add ARM32/Thumb/`armeabi-v7a` work unless an approved plan explicitly changes scope.
3. **Interpreter before JIT.** The interpreter is the semantic reference. JIT changes require differential tests.
4. **W^X always.** Do not retain writable-and-executable code pages.
5. **No silent fallback.** Unsupported ABI, ELF, relocation, instruction, or runtime behavior must yield structured diagnostics.
6. **No undocumented architecture changes.** Update the relevant design document and execution plan in the same change.

## Where to start

| Need | Read / update |
|---|---|
| Project map and dependency rules | `ARCHITECTURE.md` |
| Principles and decision rules | `docs/design-docs/core-beliefs.md` |
| Documentation index and status | `docs/design-docs/index.md` |
| Active execution work | `docs/exec-plans/active/arm64-native-bridge-r0-r1.md` |
| R0 bringup procedure and evidence checklist | `docs/aosp-hyperv-bringup.md` |
| Full long-range Native Bridge plan | `docs/arm64-native-bridge-harness-plan.md` |
| Quality gates | `docs/QUALITY_SCORE.md` |
| Reliability requirements | `docs/RELIABILITY.md` |
| Security requirements | `docs/SECURITY.md` |

## Agent workflow

1. Read this file, `ARCHITECTURE.md`, and the active execution plan.
2. Verify repository state and relevant tests before editing.
3. Keep each PR narrow: one capability, its tests, diagnostics, and required document updates.
4. Run the acceptance commands named by the active plan.
5. Record evidence and decisions in the plan before requesting review.
6. If a task exposes a missing tool, test, invariant, or document, add that enabling capability first rather than guessing.

## Documentation rules

- `AGENTS.md` is only a map; put durable details in versioned files under `docs/`.
- Plans are first-class artifacts: active plans live in `docs/exec-plans/active/`; completed plans move to `docs/exec-plans/completed/`.
- A behavior change without a test and a diagnostic path is incomplete.
- A new invariant should be mechanically enforced where practical (test, lint, CI check, or script), not only stated in prose.

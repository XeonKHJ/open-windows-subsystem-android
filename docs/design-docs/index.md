# Design Documentation Index

This directory is the durable, versioned record for design decisions. Agents should navigate from `AGENTS.md` to this index, then read only the documents relevant to their task.

| Document | Purpose | Status |
|---|---|---|
| `core-beliefs.md` | Non-negotiable engineering principles and decision rules | Active |
| `../arm64-native-bridge-harness-plan.md` | Long-range R0–R5 Native Bridge research roadmap | Active |
| `../exec-plans/active/arm64-native-bridge-r0-r1.md` | Executable short-horizon plan for the current milestone | Active |
| `../QUALITY_SCORE.md` | Measurable quality gates and scorecard | Active |
| `../RELIABILITY.md` | Test, repeatability, diagnostics, and recovery requirements | Active |
| `../SECURITY.md` | Security invariants for ELF loading, generated code, and logs | Active |

## Status vocabulary

- **Draft** — proposed, not approved for implementation.
- **Active** — currently governs implementation.
- **Superseded** — retained for history; links to its replacement.
- **Completed** — the associated work passed its acceptance gate.

## Maintenance rule

Any PR that changes an architectural boundary, acceptance criterion, execution sequence, or security invariant must update this index or the linked governing document in the same PR.

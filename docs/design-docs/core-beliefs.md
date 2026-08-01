# Core Beliefs

## 1. Humans set intent; agents execute bounded work

Human effort should concentrate on product direction, architecture, acceptance criteria, and judgment calls. Agents should implement, test, inspect, document, and iterate inside explicit constraints.

## 2. The repository is the record system

A design decision that exists only in chat, an issue comment, or a maintainer's memory is unavailable to future agents. Durable decisions, plans, acceptance evidence, and known limitations belong in version-controlled repository artifacts.

## 3. A map beats a manual

Keep `AGENTS.md` short and navigational. Place deep information in indexed documents close to the governed code or plan. Prefer progressive disclosure over one large instruction file.

## 4. Depth before breadth

Unlock the next capability by completing a narrow vertical slice, including tests and diagnostics. For Native Bridge: prove adapter control flow before loader work; prove interpreter correctness before JIT; prove controlled libraries before real applications.

## 5. Constraints are leverage

Important properties must be mechanically checked where possible. Examples: no ARM32 scope creep, W^X for generated code, structured diagnostics, interpreter/JIT differential tests, and documented execution-plan gates.

## 6. Evidence is part of the feature

A claim such as “ARM64 JNI works” is incomplete without a reproducible command, expected result, logs, and a failure classification. Evidence is committed or referenced by a committed plan.

## 7. Fail closed and explain why

Unsupported instructions, malformed ELF files, unresolved symbols, unsafe relocations, and unimplemented runtime behavior must stop in a controlled way. They must never silently produce an incorrect result or fall back to an unrelated ABI.

## 8. Optimize for future-agent readability

Prefer explicit contracts, stable interfaces, small modules, predictable paths, structured logs, and testable behavior over cleverness. A future agent must be able to infer the system from repository-local evidence.

## 9. Entropy requires continuous cleanup

When a review finds repeated ambiguity, drift, boilerplate, or a risky pattern, turn the lesson into a document update, test, script, lint, or CI check. Do not rely on repeated reminders.

## 10. Autonomy is earned by feedback loops

Broader agent autonomy requires reliable build/test commands, reproducible environments, observability, narrow plans, reviewable PRs, and recovery paths. Increase autonomy only after these controls are demonstrated.

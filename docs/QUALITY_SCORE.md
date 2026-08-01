# Quality Score

This scorecard makes the current research stage legible to humans and agents. It is not a marketing score. A stage may not advance when a mandatory gate is red.

## Score dimensions

| Dimension | Measurement | R0.0 target | R0.1 target | R0.2 target | R1 target | Owner evidence |
|---|---|---:|---:|---:|---:|---|
| Reproducibility | Builds and VM snapshots produce consistent results | Two clean builds with matching image hashes | 9/10 clean-boot attempts reach `sys.boot_completed=1` | 9/10 x86_64 JNI smoke runs return `42` | 100 adapter runs without process crash | run summary + environment baseline |
| Testability | Required commands are scripted and deterministic | Build and image-hash comparison commands | Boot-attempt sequence with reset procedure | One-command build/install/run/collect/reset workflow | ARM64 adapter integration test | CI/local command output |
| Observability | Required structured fields emitted on success/failure | Environment fingerprint; image hashes | Boot log, init log, logcat, Hyper-V events; failure category | run_id artifact dir with logcat, tombstone, result.json; failure category | bridge, ABI, library, symbol, error class | captured diagnostic record |
| Architectural conformance | Dependency and scope rules are respected | Build target is recorded as development baseline, not Hyper-V product | Boot-chain design documented before boot claims | ADB method and x86_64-only APK variant documented | adapter boundary documented and tested | review + static check where available |
| Security baseline | No known prohibited memory or loader behavior | no unsafe build shortcuts | no kernel-level security bypasses | no unsafe test shortcuts | malformed ELF and unsupported ABI fail closed | security checklist |
| Documentation freshness | Active plans and architecture match code | environment-baseline.md and hyperv-boot-design.md present | boot-design and baseline updated | test-protocol.md present | integration contract present | document links in PR |

## Mandatory red lines

The score is **red** regardless of other progress when any condition below is true:

- no reproducible test command exists for the claimed capability;
- a native crash is unclassified;
- an unsupported instruction or ABI silently returns a normal application value;
- a generated code page remains RWX;
- the implementation has moved beyond the active execution plan without a plan update;
- a change relies on disabling SELinux or weakening memory protection as its normal path;
- R1 work has started before R0.0, R0.1, and R0.2 gates are all marked complete;
- boot success is claimed without a documented Hyper-V boot-chain design.

## Review checklist

For every PR, record:

- [ ] Capability claimed
- [ ] Commands run
- [ ] Expected and observed output
- [ ] Failure-path test
- [ ] Diagnostics sample or location
- [ ] Documentation updated
- [ ] Quality score effect
- [ ] Follow-up debt, if any

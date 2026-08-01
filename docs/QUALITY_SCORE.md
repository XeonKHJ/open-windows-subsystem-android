# Quality Score

This scorecard makes the current research stage legible to humans and agents. It is not a marketing score. A stage may not advance when a mandatory gate is red.

## Score dimensions

| Dimension | Measurement | R0 target | R1 target | Owner evidence |
|---|---|---:|---:|---|
| Reproducibility | Clean VM snapshot can run the required scenario | 9/10 successful runs | 100 adapter runs without process crash | run summary + environment baseline |
| Testability | Required commands are scripted and deterministic | x86_64 JNI smoke test | ARM64 adapter integration test | CI/local command output |
| Observability | Required structured fields emitted on success/failure | boot, ADB, APK, ART/native category | bridge, ABI, library, symbol, error class | captured diagnostic record |
| Architectural conformance | Dependency and scope rules are respected | directory/plan checks | adapter boundary documented and tested | review + static check where available |
| Security baseline | No known prohibited memory or loader behavior | no unsafe test shortcuts | malformed ELF and unsupported ABI fail closed | security checklist |
| Documentation freshness | Active plans and architecture match code | baseline present | integration contract present | document links in PR |

## Mandatory red lines

The score is **red** regardless of other progress when any condition below is true:

- no reproducible test command exists for the claimed capability;
- a native crash is unclassified;
- an unsupported instruction or ABI silently returns a normal application value;
- a generated code page remains RWX;
- the implementation has moved beyond the active execution plan without a plan update;
- a change relies on disabling SELinux or weakening memory protection as its normal path.

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

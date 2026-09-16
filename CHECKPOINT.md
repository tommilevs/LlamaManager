# LlamaManager prerelease checkpoint

This branch preserves the validated `v0.3.0.dev7` field checkpoint.

- Version: `0.3.0.dev7`
- Release ZIP SHA256: `2073d10a01cfaf68e751edc5c251454e2a18a416b1565609fdb1980f1e885169`
- Verification at release: 287 tests passed; compileall, JSON/YAML, shell syntax, ZIP integrity and unpacked-release tests passed.
- Field validation: remote inspect works; active-runtime conflict remains an application conflict instead of becoming a false 503; runtime process start telemetry reconciles with systemd after reboot.
- Network note: the field environment needed a persistent LXC MTU 1400 workaround for an external PMTU black-hole. This is infrastructure-specific, not a LlamaManager application workaround.
- Next intended development area: remote benchmark execution + persistence + benchmark summaries for Brain placement decisions.

The complete `v0.3.0.dev7` source snapshot is stored directly in this branch. The branch was created as a recovery/prerelease checkpoint so development can resume from this exact source state without relying on local worktrees or chat history.

`main` is intentionally untouched.

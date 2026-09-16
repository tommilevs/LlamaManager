# LlamaManager prerelease checkpoint

This branch preserves the validated `v0.3.0.dev7` field checkpoint.

- Version: `0.3.0.dev7`
- Release ZIP SHA256: `2073d10a01cfaf68e751edc5c251454e2a18a416b1565609fdb1980f1e885169`
- Source checkpoint archive SHA256: `3a7e7ac0536111dc61117e8966ce4a406d6c358f51ef53aa4f6eea4435ea451e`
- Verification at release: 287 tests passed; compileall, JSON/YAML, shell syntax, ZIP integrity and unpacked-release tests passed.
- Field validation: remote inspect works; active-runtime conflict remains an application conflict instead of becoming a false 503; runtime process start telemetry reconciles with systemd after reboot.
- Network note: the field environment needed a persistent LXC MTU 1400 workaround for an external PMTU black-hole. This is infrastructure-specific, not a LlamaManager application workaround.
- Next intended development area: remote benchmark execution + persistence + benchmark summaries for Brain placement decisions.

The source snapshot is stored on this branch as `LlamaManager-v0.3.0.dev7-source.tar.gz`.

Restore:

```bash
sha256sum -c LlamaManager-v0.3.0.dev7-source.tar.gz.sha256
mkdir LlamaManager-v0.3.0.dev7-source
 tar -xzf LlamaManager-v0.3.0.dev7-source.tar.gz -C LlamaManager-v0.3.0.dev7-source
```

`main` is intentionally untouched; this is a prerelease recovery checkpoint.

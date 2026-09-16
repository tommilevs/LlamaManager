# LlamaManager 0.2 MCP and Localization Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an optional, secure and vendor-neutral MCP control surface to LlamaManager, complete built-in Russian localization, and add a temporary Developer Mode that exposes localization editing tools only when explicitly enabled.

**Architecture:** MCP is a thin transport/policy layer over existing LlamaManager service functions. The native HTTP MCP server runs unprivileged and reaches privileged mutations only through bounded Agent Unix-socket endpoints. Tool registration is capability-based: read-only/full-control access gates, per-operation dangerous flags, and a separate expiring Developer Mode for localization tools. Localization stays file-based and machine-local; English remains the source/default catalog with Russian built in.

**Tech Stack:** Python 3.11+, FastAPI/Starlette, `mcp>=2,<3`, HTTPX, JSON locale catalogs, PyYAML, pytest, systemd.

**Spec:** `docs/superpowers/specs/2026-09-15-llamamanager-mcp-theme-design.md`

## Global Constraints

- MCP dependency range is `mcp>=2,<3`.
- Native MCP HTTP runs as `User=llamamanager`, not root.
- Default MCP config is disabled, loopback-only `127.0.0.1:8091`, access `read-only`, auth `token`.
- Auth modes are exactly `token` and `none`; token mode remains the default and `none` emits a warning.
- Access modes are exactly `read-only` and `full-control`.
- No generic shell, arbitrary filesystem, arbitrary systemd or generic privileged execution tool exists.
- Mutating tools require `full-control`; disruptive mutations additionally require explicit dangerous flags.
- Raw MCP/Hugging Face secrets never appear in shared config, resources or audit logs.
- Developer Mode is off by default, time-limited, group-scoped, and never bypasses normal access/dangerous policy.
- English is source/default locale; Russian ships built in and missing keys fall back to English.

---

### Task 1: MCP config and authorization policy

**Files:**
- Create: `src/llamamanager/mcp_config.py`
- Test: `tests/test_mcp_config.py`

**Interfaces:**
- `MCPDangerousFlags`
- `DeveloperModeConfig`
- `MCPConfig`
- `MCPConfigManager.load() -> MCPConfig`
- `MCPConfigManager.save(config: MCPConfig) -> None`
- `MCPConfigManager.tool_allowed(tool: MCPToolDefinition) -> bool`
- `MCPConfigManager.developer_group_enabled(group: str, now: datetime | None = None) -> bool`

- [ ] **Step 1: Write failing defaults tests**

```python
def test_mcp_defaults_are_secure(tmp_path):
    manager = MCPConfigManager(tmp_path / "config.yaml")
    cfg = manager.load()
    assert cfg.enabled is False
    assert cfg.bind_host == "127.0.0.1"
    assert cfg.port == 8091
    assert cfg.endpoint == "/mcp"
    assert cfg.access == "read-only"
    assert cfg.auth == "token"
    assert cfg.developer.enabled is False
```

- [ ] **Step 2: Run RED**

```bash
pytest tests/test_mcp_config.py -q
```

- [ ] **Step 3: Implement dataclasses, YAML persistence and capability predicates**

Dangerous flags are explicit booleans. Developer mode stores only local config:

```yaml
developer:
  enabled: false
  expires_at: null
  groups: []
```

- [ ] **Step 4: Run GREEN**

```bash
pytest tests/test_mcp_config.py -q
```

- [ ] **Step 5: Commit**

```bash
git add src/llamamanager/mcp_config.py tests/test_mcp_config.py
git commit -m "feat: add MCP policy configuration"
```

### Task 2: Token digest and audit log

**Files:**
- Create: `src/llamamanager/security.py`
- Create: `src/llamamanager/audit.py`
- Test: `tests/test_security.py`
- Test: `tests/test_audit.py`

**Interfaces:**
- `generate_token() -> str`
- `hash_token(token: str) -> str`
- `store_token_digest(path: Path, digest: str) -> None`
- `verify_token(token: str, digest: str) -> bool`
- `AuditLogger.log_tool_call(...) -> None`

- [ ] **Step 1: Test digest-only storage and redaction**

```python
def test_token_store_contains_digest_not_raw_token(tmp_path):
    token = generate_token()
    digest = hash_token(token)
    path = tmp_path / "token.sha256"
    store_token_digest(path, digest)
    assert token not in path.read_text()
    assert path.stat().st_mode & 0o777 == 0o600
```

Audit assertions verify keys containing `token`, `secret`, `password` are redacted.

- [ ] **Step 2: RED**

- [ ] **Step 3: Implement helpers with `secrets.token_urlsafe`, SHA-256 and `hmac.compare_digest`**

- [ ] **Step 4: GREEN**

- [ ] **Step 5: Commit**

### Task 3: Bounded Agent mutation endpoints

**Files:**
- Modify: `src/llamamanager/agent.py`
- Modify: `src/llamamanager/agent_client.py`
- Test: `tests/test_agent_mcp.py`

**Interfaces:** explicit build/HF/service/i18n mutations only.

- [ ] **Step 1: Add failing endpoint tests and negative tests for no generic shell/systemd/path API**

- [ ] **Step 2: RED**

- [ ] **Step 3: Implement bounded Pydantic request models and handlers over existing managers**

- [ ] **Step 4: GREEN**

- [ ] **Step 5: Commit**

### Task 4: MCP registry, resources and prompts

**Files:**
- Create: `src/llamamanager/mcp_server.py`
- Test: `tests/test_mcp_server.py`

**Interfaces:**
- `build_mcp_registry(config, services, agent_client)`
- read-only tool set;
- full-control filtered tool set;
- safe resources and prompts.

- [ ] **Step 1: Test read-only registry**

Expected tools include hardware, builds, models, Hugging Face, benchmark history, service status, config summary and cluster read tools. Resources exclude raw secrets.

- [ ] **Step 2: Test authorization filtering**

Mutations absent from read-only; dangerous tools absent unless their specific flag is true.

- [ ] **Step 3: RED**

- [ ] **Step 4: Implement tool definitions and wrappers**

- [ ] **Step 5: GREEN and commit**

### Task 5: Localization service and temporary Developer Mode

**Files:**
- Create: `src/llamamanager/i18n_service.py`
- Add: `src/llamamanager/locales/metadata/en.json`
- Add: `src/llamamanager/locales/metadata/ru.json`
- Modify: locale catalogs
- Test: `tests/test_i18n_service.py`
- Test: `tests/test_mcp_server.py`

**Interfaces:**
- `get_supported_locales()`
- `get_translation_catalog(locale)`
- `list_missing_translation_keys(locale)`
- `upsert_translation_entries(locale, entries)`

- [ ] **Step 1: Test supported locales and missing-key report**

- [ ] **Step 2: Test atomic allowed writes and rejection of unknown source keys**

- [ ] **Step 3: Test MCP tools absent until Developer Mode is enabled and unexpired**

- [ ] **Step 4: RED**

- [ ] **Step 5: Implement and GREEN**

- [ ] **Step 6: Commit**

### Task 6: MCP CLI and runtime

**Files:**
- Modify: `src/llamamanager/cli.py`
- Modify: `src/llamamanager/mcp_server.py`
- Test: `tests/test_cli.py`

**Interfaces:**
- `llamarun mcp status`
- `llamarun mcp config show/set`
- `llamarun mcp token regenerate`
- `llamarun mcp enable/disable`
- `llamarun mcp tools/resources`
- `llamarun mcp developer enable --for DURATION --group NAME`
- `llamarun mcp developer disable`
- `llamarun mcp stdio`

- [ ] **Step 1: Test parser and redacted status**

- [ ] **Step 2: Test `auth none` warning and Developer Mode duration parsing**

- [ ] **Step 3: RED**

- [ ] **Step 4: Implement CLI/runtime wiring**

- [ ] **Step 5: GREEN and commit**

### Task 7: Native MCP service and installer

**Files:**
- Add: `systemd/llamamanager-mcp.service`
- Modify: `scripts/install.sh`
- Modify: `scripts/uninstall.sh`
- Modify: `scripts/config.example.yaml`
- Test: `tests/test_installer.py`

**Interfaces:** unprivileged native HTTP service.

- [ ] **Step 1: Test unit contains `User=llamamanager`, `Group=llamamanager`, no root, disabled-by-default install behavior**

- [ ] **Step 2: RED**

- [ ] **Step 3: Add systemd/install integration**

- [ ] **Step 4: GREEN**

- [ ] **Step 5: Commit**

### Task 8: Complete Russian localization and Web locale UX

**Files:**
- Modify: `src/llamamanager/locales/en.json`
- Modify: `src/llamamanager/locales/ru.json`
- Modify: `src/llamamanager/web.py`
- Modify: `src/llamamanager/templates/base.html`
- Test: `tests/test_i18n_completeness.py`
- Test: `tests/test_i18n_web.py`

**Interfaces:** EN/RU key parity and browser locale selector.

- [ ] **Step 1: Add parity and rendered-language tests**

- [ ] **Step 2: RED**

- [ ] **Step 3: Fill RU catalog and locale selector/cookie path**

- [ ] **Step 4: GREEN and commit**

### Task 9: Documentation and release gate

**Files:**
- Modify: `README.md`
- Modify: `docs/MCP.md`
- Modify: `docs/I18N.md`
- Modify: `VERSION`

**Interfaces:** operator setup/security/developer workflow docs.

- [ ] **Step 1: Update documentation**

- [ ] **Step 2: Full verification**

```bash
python -m pytest -q
python -m compileall -q src
```

- [ ] **Step 3: Package and inspect release ZIP/checksum**

- [ ] **Step 4: Commit only after clean verification**

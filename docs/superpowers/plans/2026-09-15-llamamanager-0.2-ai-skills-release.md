# LlamaManager 0.2 AI Skills and Release Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend LlamaManager 0.2 with a local opt-in AI Skill Pack (MCP) over the existing domain services, complete built-in Russian localization with safe temporary translation editing, apply the new Web visual system, and ship a verified installable prerelease.

**Architecture:** MCP is a thin unprivileged transport/policy layer over existing LlamaManager service functions. The HTTP MCP service runs as `llamamanager` and reaches privileged mutations only through the Agent Unix socket. Tool registration is capability-filtered by read-only/full-control access, per-operation dangerous flags, and a separate time-limited Developer Mode for localization. The Web UI consumes the same domain/Agent contracts and uses a server-rendered tokenized theme system with browser-local theme state.

**Tech Stack:** Python 3.11+, FastAPI/Starlette, `mcp>=2,<3`, HTTPX, PyYAML, Jinja2, JSON locale catalogs, SQLite, Bash/systemd, Docker Compose, pytest.

**Spec:** `docs/superpowers/specs/2026-09-15-llamamanager-mcp-theme-design.md`

## Global Constraints

- Python runtime floor is 3.11+ and MCP dependency stays `mcp>=2,<3`.
- MCP is optional, installed but disabled by default, and HTTP binds `127.0.0.1:8091` unless explicitly changed.
- MCP auth modes are exactly `token` and `none`; `token` is the default.
- MCP access modes are exactly `read-only` and `full-control`; `read-only` is the default.
- No generic shell, arbitrary filesystem, or arbitrary systemd tools are exposed.
- Raw MCP/Hugging Face secrets never appear in shared config, MCP resources, or audit logs.
- The native MCP service runs unprivileged as `llamamanager`; privileged mutations use only bounded Agent Unix-socket endpoints.
- Developer Mode is off by default, time-limited, group-scoped, and never bypasses normal access/dangerous checks.
- English is the source/default locale and Russian ships built in.
- Web themes are Auto/Light/Dark; theme choice is browser-local and must not rewrite machine YAML.
- The release artifact must include docs, systemd assets, Docker assets, translations, MCP runtime code and tests needed for reproducible verification.

---

### Task 1: MCP configuration and authorization policy

**Files:**
- Create: `src/llamamanager/mcp_config.py`
- Modify: `pyproject.toml`
- Test: `tests/test_mcp_config.py`

**Interfaces:**
- `MCPDangerousFlags`
- `MCPConfig`
- `DeveloperModeConfig`
- `MCPConfigManager.load() -> MCPConfig`
- `MCPConfigManager.save(config: MCPConfig) -> None`
- `MCPConfigManager.should_register(tool: ToolDefinition) -> bool`
- `MCPConfigManager.is_developer_enabled(group: str) -> bool`

- [ ] **Step 1:** Write failing tests for defaults: disabled, loopback bind, port `8091`, endpoint `/mcp`, access `read-only`, auth `token`, all dangerous flags false, Developer Mode off.
- [ ] **Step 2:** Run `pytest tests/test_mcp_config.py -q` and confirm expected failures.
- [ ] **Step 3:** Implement config dataclasses, YAML round-trip and authorization predicates.
- [ ] **Step 4:** Run targeted tests and confirm pass.
- [ ] **Step 5:** Commit.

### Task 2: MCP token authentication and audit

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

- [ ] **Step 1:** Test raw token is returned once, only digest persisted, file mode `0600`, constant-time compare path, redacted audit args.
- [ ] **Step 2:** Run RED.
- [ ] **Step 3:** Implement minimal token/audit helpers.
- [ ] **Step 4:** Run GREEN.
- [ ] **Step 5:** Commit.

### Task 3: Agent bounded mutation endpoints

**Files:**
- Modify: `src/llamamanager/agent.py`
- Modify: `src/llamamanager/agent_client.py`
- Test: `tests/test_agent_mcp.py`

**Interfaces:**
- explicit bounded endpoints for build install/update/build, HF download/token, service restart, localization update and cluster trust operations;
- `AgentClient` methods matching the bounded surface.

- [ ] **Step 1:** Add failing endpoint/contract tests; assert no generic shell/systemd/path endpoint exists.
- [ ] **Step 2:** Run RED.
- [ ] **Step 3:** Add bounded request models and endpoint handlers reusing domain services.
- [ ] **Step 4:** Run GREEN.
- [ ] **Step 5:** Commit.

### Task 4: MCP registry and read-only tools/resources/prompts

**Files:**
- Create: `src/llamamanager/mcp_server.py`
- Test: `tests/test_mcp_server.py`

**Interfaces:**
- `build_mcp_registry(config, services, agent_client) -> Registry`
- read-only tools, safe resources, guidance prompts.

- [ ] **Step 1:** Test standard read-only inventory/search/status tools, resources and prompts are present and secrets are absent.
- [ ] **Step 2:** Run RED.
- [ ] **Step 3:** Implement registry/tool wrappers over domain services.
- [ ] **Step 4:** Run GREEN.
- [ ] **Step 5:** Commit.

### Task 5: Full-control and dangerous gating

**Files:**
- Modify: `src/llamamanager/mcp_server.py`
- Test: `tests/test_mcp_server.py`

**Interfaces:** same registry filtered by policy.

- [ ] **Step 1:** Test mutations absent in read-only, present only in full-control, disruptive tools require matching dangerous flags.
- [ ] **Step 2:** RED.
- [ ] **Step 3:** Implement capability filter.
- [ ] **Step 4:** GREEN.
- [ ] **Step 5:** Commit.

### Task 6: Temporary localization Developer Mode

**Files:**
- Modify: `src/llamamanager/mcp_config.py`
- Modify: `src/llamamanager/mcp_server.py`
- Modify: `src/llamamanager/i18n_service.py`
- Add: `src/llamamanager/locales/metadata/en.json`
- Add: `src/llamamanager/locales/metadata/ru.json`
- Test: `tests/test_i18n_service.py`
- Test: `tests/test_mcp_server.py`

**Interfaces:**
- `get_supported_locales`
- `get_translation_catalog`
- `list_missing_translation_keys`
- `upsert_translation_entries`

- [ ] **Step 1:** Test tools absent by default, appear only while enabled, writes require full-control, writes are atomic, unknown English key rejected for non-English locale.
- [ ] **Step 2:** RED.
- [ ] **Step 3:** Implement metadata/catalog helpers and Developer Mode tool registration.
- [ ] **Step 4:** GREEN.
- [ ] **Step 5:** Commit.

### Task 7: CLI management and MCP runtime

**Files:**
- Modify: `src/llamamanager/cli.py`
- Modify: `src/llamamanager/mcp_server.py`
- Test: `tests/test_cli.py`
- Test: `tests/test_mcp_server.py`

**Interfaces:**
- `llamarun mcp status/config/token/enable/disable/tools/resources/developer`
- `llamarun mcp stdio`
- `llamamanager-mcp` HTTP entry point.

- [ ] **Step 1:** Test CLI parsing, status redaction, no-auth warning, Developer Mode duration/group semantics, stdio path.
- [ ] **Step 2:** RED.
- [ ] **Step 3:** Implement CLI and runtime entry points.
- [ ] **Step 4:** GREEN.
- [ ] **Step 5:** Commit.

### Task 8: Web visual redesign and localization completion

**Files:**
- Modify: `src/llamamanager/templates/*.html`
- Modify: `src/llamamanager/static/app.css`
- Modify: `src/llamamanager/static/theme.js`
- Modify: `src/llamamanager/locales/en.json`
- Modify: `src/llamamanager/locales/ru.json`
- Modify: `src/llamamanager/web.py`
- Test: `tests/test_i18n_web.py`
- Test: `tests/test_i18n_completeness.py`
- Test: `tests/test_web.py`

**Interfaces:**
- browser-local theme selector; server-rendered localized pages.

- [ ] **Step 1:** Test EN/RU key parity and rendering, theme buttons/JS presence, no config-write on theme change.
- [ ] **Step 2:** RED.
- [ ] **Step 3:** Complete translations and visual system.
- [ ] **Step 4:** GREEN.
- [ ] **Step 5:** Commit.

### Task 9: Native MCP service and installer integration

**Files:**
- Add: `systemd/llamamanager-mcp.service`
- Modify: `scripts/install.sh`
- Modify: `scripts/uninstall.sh`
- Modify: `scripts/config.example.yaml`
- Test: installer tests/shell checks.

**Interfaces:** unprivileged service lifecycle, config/token permissions.

- [ ] **Step 1:** Add assertions for service User/Group, disabled-by-default installer state, config/token dirs.
- [ ] **Step 2:** RED.
- [ ] **Step 3:** Implement unit/install integration.
- [ ] **Step 4:** GREEN.
- [ ] **Step 5:** Commit.

### Task 10: Documentation and prerelease packaging

**Files:**
- Modify: `README.md`
- Modify: `docs/MCP.md`
- Modify: `docs/I18N.md`
- Modify: `docs/DOCKER.md`
- Modify: `VERSION`
- Modify: release script if required.

**Interfaces:** install, MCP setup, security model, developer workflow, version metadata.

- [ ] **Step 1:** Update docs and version to the chosen 0.2 prerelease.
- [ ] **Step 2:** Run full pytest and compile checks.
- [ ] **Step 3:** Run release script, inspect ZIP and checksum.
- [ ] **Step 4:** Commit and tag only after all verification passes.

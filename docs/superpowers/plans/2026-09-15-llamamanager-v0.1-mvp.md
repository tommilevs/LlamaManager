# LlamaManager v0.1 MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an installable Linux-first LlamaManager MVP that unifies llama.cpp forks, hardware profiles, shared/local GGUF discovery, presets, Hugging Face search/download, Agent/Web control, persistence, migration, systemd and a Docker Web UI.

**Architecture:** YAML is the source of truth for portable configuration. Machine-local YAML overlays shared source/preset data and extensible hardware/build recipes; SQLite stores only rebuildable indexes and benchmark history. FastAPI powers the local Agent and Web layers, with the Agent designed for a Unix socket boundary.

**Tech Stack:** Python 3.11+, PyYAML, huggingface_hub, FastAPI, Uvicorn, HTTPX, Jinja2, SQLite, Bash, systemd, Docker Compose.

**Spec:** `docs/SPEC-v0.1.md`

## Global Constraints

- English is the source/default locale; Russian ships built in.
- Built-in immutable hardware strategies are `auto`, `vulkan`, `cpu`, `cuda`; `vulcan` is an alias only.
- External hardware profiles initially include `rx5500xt`, `rtx4090`, `rtx5060`, `tesla-v100-32gb`.
- Builds live under `/opt/llama/builds`; generated model view under `/opt/llama/models`; app under `/opt/llama/llamamanager`.
- Model resolver priority is `local > shared` and must never delete unmanaged physical model files.
- Dirty Git repositories are never reset automatically.
- Hugging Face downloads are selective: chosen quant shards plus selected companions only.
- Secrets are never stored in shared YAML and token files are mode `0600`.
- Native and Docker Web UIs use the same Agent API contract.

---

### Task 1: Configuration, Paths and i18n
**Files:** `src/llamamanager/config.py`, `src/llamamanager/i18n.py`, locale JSON, tests.
**Interfaces:** `ConfigLoader.load() -> AppConfig`; `Translator.t(key, **kwargs) -> str`.
- [ ] Write failing tests for layered config, `vulcan` alias and English fallback.
- [ ] Run targeted tests and confirm failures.
- [ ] Implement minimal config/i18n behavior.
- [ ] Run targeted and full tests.

### Task 2: Hardware Inventory and Profiles
**Files:** `src/llamamanager/hardware.py`, hardware YAML profiles, tests.
**Interfaces:** `HardwareDetector.detect() -> HardwareInventory`; `ProfileRegistry.match(device) -> str | None`.
- [ ] Test multiple NVIDIA/AMD devices and exact profile matching.
- [ ] Verify red, implement, verify green.

### Task 3: Build Registry / Git Update / Build Variants
**Files:** `src/llamamanager/builds.py`, build recipe YAML, tests.
**Interfaces:** `BuildManager.status()`, `install()`, `update()`, `build()`.
- [ ] Test recipe loading, dirty update refusal and per-profile build directory generation.
- [ ] Verify red, implement, verify green.

### Task 4: Model Resolver and Safe Generated View
**Files:** `src/llamamanager/models.py`, tests.
**Interfaces:** `ModelResolver.scan() -> list[ModelRecord]`; `rebuild_view(records)`.
- [ ] Test local-over-shared resolution, companions and unmanaged-file safety.
- [ ] Verify red, implement, verify green.

### Task 5: Preset Engine
**Files:** `src/llamamanager/presets.py`, tests.
**Interfaces:** `PresetStore.resolve(model_id, machine_id)`, `generate_ini()`.
- [ ] Test shared/local overlay and valid named INI generation.
- [ ] Verify red, implement, verify green.

### Task 6: Hugging Face Provider
**Files:** `src/llamamanager/huggingface.py`, tests.
**Interfaces:** `search_models`, `inspect_repo`, `build_download_plan`, `download_plan`.
- [ ] Test capability extraction, quant grouping, shard grouping, companion detection and token permissions.
- [ ] Verify red, implement, verify green.

### Task 7: SQLite Index and Benchmarks
**Files:** `src/llamamanager/database.py`, tests.
**Interfaces:** `StateDB.replace_models`, `search_models`, `add_benchmark`.
- [ ] Test rebuildable model index and benchmark persistence.
- [ ] Verify red, implement, verify green.

### Task 8: CLI
**Files:** `src/llamamanager/cli.py`, tests.
**Interfaces:** `llamarun` console command with hardware/models/builds/presets/HF/serve subcommands.
- [ ] Test parser and non-destructive status/list flows.
- [ ] Verify red, implement, verify green.

### Task 9: Agent and Web
**Files:** `src/llamamanager/agent.py`, `web.py`, templates, tests.
**Interfaces:** FastAPI Agent endpoints `/health`, `/hardware`, `/models`, `/builds`; Web dashboard/search.
- [ ] Test health/models API and Web page render.
- [ ] Verify red, implement, verify green.

### Task 10: Installer, Migration, systemd, Docker and Release
**Files:** scripts, units, Docker files, docs, release tooling.
**Interfaces:** `install.sh`, `migrate_layout()`, service units, release archive.
- [ ] Test migration planning without destructive moves.
- [ ] Implement installer/migration/systemd/docker assets.
- [ ] Run full pytest, compileall, YAML/JSON validation, CLI smoke checks.
- [ ] Package ZIP and SHA256 and inspect archive content.

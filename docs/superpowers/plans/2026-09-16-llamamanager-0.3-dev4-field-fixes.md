# LlamaManager 0.3.0.dev4 Field Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the concrete issues found during the first physical two-node LlamaManager field test and ship a verified `0.3.0.dev4` artifact.

**Architecture:** Keep the dedicated `llamacluster` privilege boundary. Root/Agent remains responsible for config migration and secret creation; the unprivileged Cluster Gateway reads normalized config without persisting migrations. mDNS advertises resolvable IP addresses in DNS A/AAAA data while keeping TXT restricted to the existing non-secret allowlist. Hardware/host inspection fixes remain small, deterministic normalization changes.

**Tech Stack:** Python 3.12, FastAPI, zeroconf, pytest, Bash/systemd.

**Spec:** Field evidence from the 2026-09-16 `.32 ↔ .33` two-node test plus `docs/superpowers/specs/2026-09-16-llamamanager-join-uri-pairing-design.md` and `docs/superpowers/specs/2026-09-16-llamamanager-peer-endpoints-design.md`.

## Global Constraints

- Preserve `User=llamacluster`, `Group=llamacluster`, `SupplementaryGroups=llamamanager` for the Cluster Gateway.
- Do not make the config directory writable by the Cluster Gateway.
- Keep discovery explicitly untrusted; never publish Join URI, ticket, credential or peer secrets through mDNS.
- Preserve existing peer identities and credentials across upgrade.
- Do not change the pairing cryptographic transcript/protocol in this fix release.
- Use TDD RED → GREEN for every behavior change.

---

### Task 1: Cluster privilege boundary and config reads

**Files:**
- Modify: `scripts/install.sh`
- Modify: `src/llamamanager/cluster_config.py`
- Modify: `src/llamamanager/cluster_server.py`
- Modify: `src/llamamanager/cluster_discovery.py`
- Test: `tests/test_installer.py`
- Test: `tests/test_cluster_config.py`
- Test: `tests/test_cluster_server.py`

**Interfaces:**
- `ClusterConfigManager.load(*, persist_migrations: bool = True) -> ClusterConfig`
- `ClusterConfigManager.peer(node_id, *, persist_migrations: bool = True) -> PeerRecord | None`
- Cluster Gateway and `LanAdvertiser` call these with `persist_migrations=False`.

- [ ] Add failing installer test requiring parent `secrets` ownership `root:llamacluster` and mode `0710` while `secrets/cluster` remains `llamacluster:llamacluster` `0700`.
- [ ] Run the installer test and verify RED.
- [ ] Update installer permissions minimally and verify GREEN.
- [ ] Add failing config test proving a legacy schema can be normalized in memory with `persist_migrations=False` while the source file bytes remain unchanged.
- [ ] Run the config test and verify RED.
- [ ] Implement non-persisting load/peer path and verify GREEN.
- [ ] Add server/advertiser tests proving the unprivileged paths use non-persisting reads.
- [ ] Run relevant tests and commit.

### Task 2: Resolvable mDNS advertisements

**Files:**
- Modify: `src/llamamanager/cluster_discovery.py`
- Test: `tests/test_cluster_discovery.py`

**Interfaces:**
- `DiscoveryRecord.addresses: tuple[str, ...]`
- `ZeroconfDiscoveryAdapter(address_provider: Callable[[], Iterable[str]] | None = None)`
- ServiceInfo receives packed IPv4/IPv6 `addresses` while TXT remains unchanged.

- [ ] Add failing test proving `_make_info()` includes packed LAN IP addresses and that TXT keys remain the existing strict allowlist.
- [ ] Add failing test proving `LanAdvertiser` takes a literal IP from `advertise_address` into the discovery record.
- [ ] Run tests and verify RED.
- [ ] Implement filtered local address collection plus configured-address hint, deduplication, and packing.
- [ ] Verify targeted discovery tests GREEN.
- [ ] Commit.

### Task 3: Hardware and host inspection normalization

**Files:**
- Modify: `src/llamamanager/hardware.py`
- Modify: `src/llamamanager/host_inspect.py`
- Test: `tests/test_hardware.py`
- Test: `tests/test_host_inspect.py`

**Interfaces:**
- Vulkan vendor normalization recognizes Intel, AMD and NVIDIA names.
- Host OS uses `os.uname().sysname`, with compatibility fallback to `system`.

- [ ] Add failing Intel UHD 630 vendor regression test.
- [ ] Verify RED, implement minimal vendor normalization, verify GREEN.
- [ ] Add failing `os.uname_result`-shape regression test for `Linux` instead of `unknown`.
- [ ] Verify RED, implement sysname fallback, verify GREEN.
- [ ] Commit.

### Task 4: Release metadata, documentation and package gate

**Files:**
- Modify: `VERSION`
- Modify: `pyproject.toml`
- Modify: `src/llamamanager/__init__.py`
- Modify: `README.md`
- Modify: `docs/CLUSTER.md`

**Interfaces:**
- Version becomes `0.3.0.dev4`.

- [ ] Update release metadata and concise field-fix notes.
- [ ] Run full `pytest -q`.
- [ ] Run `./scripts/make-release.sh` and inspect all release-gate output.
- [ ] Verify ZIP integrity and portable checksum from the release directory.
- [ ] Copy the verified ZIP and checksum to `/mnt/data` and independently recalculate SHA256.
- [ ] Commit release metadata only after verification evidence is clean.

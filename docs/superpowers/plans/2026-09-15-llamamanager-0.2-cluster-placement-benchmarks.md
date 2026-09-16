# LlamaManager 0.2 Cluster, Placement and Benchmark Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add secure heterogeneous-node federation to LlamaManager so one Web/MCP endpoint can inspect trusted peers, reason about model placement, and collect non-disruptive benchmark evidence without requiring a local LLM.

**Architecture:** Each host remains a standalone LlamaNode with a stable local identity. Nodes form explicit pairwise trust relationships and communicate through a new unprivileged Cluster Gateway on `8092`; the Gateway reaches privileged local data/actions only through the existing Agent Unix socket. The Brain is deterministic state + rules + measurements; Codex/DeepSeek/other LLMs remain optional MCP clients above that layer.

**Tech Stack:** Python 3.11+, FastAPI/Uvicorn, HTTPX, PyYAML, SQLite, HMAC-SHA256, systemd, pytest.

**Spec:** `docs/superpowers/specs/2026-09-15-llamamanager-mcp-theme-design.md` plus the approved cluster architecture from the 2026-09-15 design session.

## Global Constraints

- Existing standalone CLI/Web/MCP behavior must remain functional when cluster mode is disabled.
- No permanent coordinator or mandatory primary/master node.
- Cluster trust is pairwise and non-transitive by default.
- The cluster Gateway runs unprivileged as `llamacluster`; privileged local inspection/actions go through the bounded Agent API.
- No generic shell, arbitrary filesystem path, arbitrary systemd unit or generic privileged execution endpoint may be introduced.
- Pairing credentials are local machine secrets; shared storage never contains raw cluster credentials.
- The stable fingerprint is an identity comparison aid derived from the stable node ID; it is not documented as a signing-certificate fingerprint.
- LAN HTTP is allowed only as a trusted-LAN first milestone; HTTPS/proxy endpoints must be representable by configuration without protocol redesign.
- Brain decisions must be deterministic and explainable from inventory/config/measurements; no local LLM dependency.
- Benchmark probes are non-disruptive by default and must not silently stop services, evict models, or take over busy accelerators.
- Cluster integration in MCP remains vendor-neutral; Codex is one possible MCP client, not a protocol requirement.

---

### Task 1: Cluster identity and configuration

**Files:**
- Create: `src/llamamanager/cluster_config.py`
- Modify: `scripts/config.example.yaml`
- Modify: `scripts/install.sh`
- Modify: `scripts/uninstall.sh`
- Test: `tests/test_cluster_config.py`
- Test: `tests/test_installer.py`

**Interfaces:**
- `ClusterIdentity(node_id: str, display_name: str)`
- `ClusterConfig(bind_host: str, port: int, enabled: bool, advertise_address: str | None, identity: ClusterIdentity, peers: tuple[PeerRecord, ...])`
- `ClusterConfigManager.load() -> ClusterConfig`
- `ClusterConfigManager.set_runtime_config(...) -> ClusterConfig`
- `ClusterConfigManager.create_identity(display_name: str | None = None) -> ClusterIdentity`

- [ ] **Step 1: Write failing identity/config tests**

```python
def test_cluster_defaults_are_loopback_disabled(tmp_path):
    manager = ClusterConfigManager(tmp_path / "cluster.yaml", tmp_path / "secrets")
    config = manager.load()
    assert config.enabled is False
    assert config.bind_host == "127.0.0.1"
    assert config.port == 8092
    assert config.peers == ()


def test_create_identity_is_stable_and_fingerprint_is_human_compare_aid(tmp_path):
    manager = ClusterConfigManager(tmp_path / "cluster.yaml", tmp_path / "secrets")
    first = manager.create_identity("llama-a")
    second = manager.create_identity("ignored-new-name")
    assert first.node_id == second.node_id
    assert first.display_name == second.display_name
    assert manager.fingerprint(first.node_id).count(":") == 7
```

- [ ] **Step 2: Run RED**

```bash
pytest tests/test_cluster_config.py tests/test_installer.py -q
```

Expected: cluster config and installer assertions fail because the new identity/config/service assets do not exist.

- [ ] **Step 3: Implement identity/config storage and installer paths**

Create cluster config under:

```text
/opt/llama/llamamanager/cluster.yaml
```

Create local-only identity/credential material under:

```text
/opt/llama/llamamanager/secrets/cluster/
```

Identity JSON is public metadata; credential files are `0600`.

- [ ] **Step 4: Run GREEN**

```bash
pytest tests/test_cluster_config.py tests/test_installer.py -q
```

- [ ] **Step 5: Commit**

```bash
git add src/llamamanager/cluster_config.py scripts/config.example.yaml scripts/install.sh scripts/uninstall.sh tests/test_cluster_config.py tests/test_installer.py
git commit -m "feat: add cluster identity and config"
```

### Task 2: Pairwise trust and credential storage

**Files:**
- Create: `src/llamamanager/cluster_auth.py`
- Modify: `src/llamamanager/cluster_config.py`
- Test: `tests/test_cluster_auth.py`
- Test: `tests/test_cluster_config.py`

**Interfaces:**
- `PeerRecord(node_id: str, name: str, address: str, fingerprint: str, credential_id: str, enabled: bool)`
- `PeerCredentialStore.issue_peer_secret(...) -> str`
- `PeerCredentialStore.load_peer_secret(...) -> str`
- `sign_cluster_request(secret, method, path, timestamp, nonce, body) -> str`
- `verify_cluster_request(...) -> bool`

- [ ] **Step 1: Write failing trust/auth tests**

```python
def test_peer_secret_file_is_mode_0600(tmp_path):
    store = PeerCredentialStore(tmp_path)
    secret = store.issue_peer_secret("node-b", "cred-1")
    path = tmp_path / "node-b.secret"
    assert path.stat().st_mode & 0o777 == 0o600
    assert secret not in (tmp_path / "cluster.yaml").read_text() if (tmp_path / "cluster.yaml").exists() else True


def test_cluster_signature_rejects_body_tamper():
    signature = sign_cluster_request("secret", "POST", "/cluster/v1/placement/recommend", 100, "abc", b"{}")
    assert verify_cluster_request("secret", signature, "POST", "/cluster/v1/placement/recommend", 100, "abc", b'{"x":1}') is False
```

- [ ] **Step 2: Run RED**

```bash
pytest tests/test_cluster_auth.py tests/test_cluster_config.py -q
```

- [ ] **Step 3: Implement HMAC request signing and peer secret storage**

Signature canonical form:

```text
METHOD\nPATH\nTIMESTAMP\nNONCE\nSHA256(BODY)
```

Secrets are random high-entropy local credentials and never enter shared YAML.

- [ ] **Step 4: Run GREEN**

```bash
pytest tests/test_cluster_auth.py tests/test_cluster_config.py -q
```

- [ ] **Step 5: Commit**

```bash
git add src/llamamanager/cluster_auth.py src/llamamanager/cluster_config.py tests/test_cluster_auth.py tests/test_cluster_config.py
git commit -m "feat: add pairwise cluster authentication"
```

### Task 3: Explicit pairing flow

**Files:**
- Create: `src/llamamanager/cluster_pairing.py`
- Modify: `src/llamamanager/cluster_server.py`
- Modify: `src/llamamanager/cluster_config.py`
- Modify: `src/llamamanager/cli.py`
- Test: `tests/test_cluster_pairing.py`
- Test: `tests/test_cli.py`

**Interfaces:**
- `PairingOffer(ticket: str, expires_at: datetime, inviter_node_id: str, inviter_name: str, inviter_fingerprint: str, inviter_address: str)`
- `PairingManager.create_offer(ttl_seconds: int = 600) -> PairingOffer`
- `PairingManager.accept_offer(remote_address: str, ticket: str, expected_fingerprint: str) -> PeerRecord`
- CLI: `llamarun cluster pair offer`, `llamarun cluster pair ADDRESS`, `llamarun cluster pair accept ...`

- [ ] **Step 1: Write failing pairing tests**

```python
def test_pairing_offer_expires(tmp_path):
    manager = PairingManager(...)
    offer = manager.create_offer(ttl_seconds=1)
    clock.advance(seconds=2)
    with pytest.raises(PairingExpired):
        manager.validate_offer(offer.ticket)


def test_pairing_requires_operator_fingerprint_confirmation(...):
    with pytest.raises(FingerprintMismatch):
        manager.accept_offer("http://node-b:8092", "ticket", "wrong")
```

- [ ] **Step 2: Run RED**

```bash
pytest tests/test_cluster_pairing.py tests/test_cli.py -q
```

- [ ] **Step 3: Implement short-lived pairing challenge and CLI workflow**

Pairing transfers public identity metadata plus newly issued peer credentials only after explicit confirmation. The acceptance path records both sides independently; no transitive trust is created.

- [ ] **Step 4: Run GREEN**

```bash
pytest tests/test_cluster_pairing.py tests/test_cli.py -q
```

- [ ] **Step 5: Commit**

```bash
git add src/llamamanager/cluster_pairing.py src/llamamanager/cluster_server.py src/llamamanager/cluster_config.py src/llamamanager/cli.py tests/test_cluster_pairing.py tests/test_cli.py
git commit -m "feat: add explicit cluster pairing"
```

### Task 4: Unprivileged Cluster Gateway and authenticated API

**Files:**
- Create: `src/llamamanager/cluster_server.py`
- Create: `systemd/llamamanager-cluster.service`
- Modify: `src/llamamanager/agent_client.py`
- Modify: `scripts/install.sh`
- Modify: `scripts/uninstall.sh`
- Test: `tests/test_cluster_server.py`
- Test: `tests/test_installer.py`

**Interfaces:**
- public: `GET /health`, `GET /cluster/v1/info`, pairing bootstrap routes;
- protected: `GET /cluster/v1/host/inspect`, `POST /cluster/v1/placement/recommend`, `POST /cluster/v1/benchmark/run`;
- no generic shell/path/systemd endpoint.

- [ ] **Step 1: Write failing Gateway tests**

```python
def test_protected_route_rejects_missing_peer_auth(test_client):
    response = test_client.get("/cluster/v1/host/inspect")
    assert response.status_code == 401


def test_cluster_service_runs_as_dedicated_unprivileged_user():
    text = Path("systemd/llamamanager-cluster.service").read_text()
    assert "User=llamacluster" in text
    assert "Group=llamacluster" in text
```

- [ ] **Step 2: Run RED**

```bash
pytest tests/test_cluster_server.py tests/test_installer.py -q
```

- [ ] **Step 3: Implement Gateway over Agent Unix socket**

Gateway request auth uses stored peer credentials, timestamp skew checks and nonce replay cache. Host inspection calls the existing Agent boundary rather than reimplementing privileged reads.

- [ ] **Step 4: Run GREEN**

```bash
pytest tests/test_cluster_server.py tests/test_installer.py -q
```

- [ ] **Step 5: Commit**

```bash
git add src/llamamanager/cluster_server.py src/llamamanager/agent_client.py systemd/llamamanager-cluster.service scripts/install.sh scripts/uninstall.sh tests/test_cluster_server.py tests/test_installer.py
git commit -m "feat: add unprivileged cluster gateway"
```

### Task 5: Host inspection resource

**Files:**
- Create: `src/llamamanager/host_inspect.py`
- Modify: `src/llamamanager/agent.py`
- Modify: `src/llamamanager/agent_client.py`
- Test: `tests/test_host_inspect.py`
- Test: `tests/test_agent.py`

**Interfaces:**
- `HostInspection` with identity, OS/kernel/arch, CPU/RAM, accelerator inventory, model storage summary, build state, managed running services.
- Agent endpoint `GET /host/inspect`.

- [ ] **Step 1: Test safe bounded inspection output**

Assert secrets, raw env and arbitrary file contents are absent.

- [ ] **Step 2: RED**

```bash
pytest tests/test_host_inspect.py tests/test_agent.py -q
```

- [ ] **Step 3: Implement local collector and Agent endpoint**

- [ ] **Step 4: GREEN**

- [ ] **Step 5: Commit**

```bash
git add src/llamamanager/host_inspect.py src/llamamanager/agent.py src/llamamanager/agent_client.py tests/test_host_inspect.py tests/test_agent.py
git commit -m "feat: add host inspection resource"
```

### Task 6: CPU-first placement and model fit

**Files:**
- Create: `src/llamamanager/placement.py`
- Modify: `src/llamamanager/cluster_server.py`
- Test: `tests/test_placement.py`

**Interfaces:**
- `PlacementRequest(model_bytes: int, backend: str = "auto", build: str | None = None, gpu_layers: int | None = None, allow_split: bool = False)`
- `recommend_placement(request, hosts) -> PlacementDecision`

- [ ] **Step 1: Write fit tests**

Include CPU-only host, insufficient VRAM unknown host, requested build missing, explicit GPU backend with no matching device.

- [ ] **Step 2: RED**

- [ ] **Step 3: Implement conservative fit rules**

First-pass reserve:

```text
required_bytes = ceil(model_bytes * 1.15)
```

CPU placement is allowed when RAM and a CPU-capable build exist. GPU placement requires explicit matching accelerator evidence; unknown VRAM is not treated as fit.

- [ ] **Step 4: GREEN**

- [ ] **Step 5: Commit**

### Task 7: Benchmark persistence and safe probes

**Files:**
- Modify: `src/llamamanager/database.py`
- Create: `src/llamamanager/benchmark_runner.py`
- Modify: `src/llamamanager/cluster_server.py`
- Test: `tests/test_database.py`
- Test: `tests/test_benchmark_runner.py`

**Interfaces:**
- `BenchmarkMeasurement(node_id, model_id, build_id, metric, value, unit, params, timestamp)`
- `run_probe(..., disruptive: bool = False)`

- [ ] **Step 1: Test persisted node/build/model identity and nondisruptive default**

Default probe may inspect hardware/history or sample a bounded file read, but must not start/stop llama processes.

- [ ] **Step 2: RED**

- [ ] **Step 3: Implement measurement storage adapter and cheap probes**

- [ ] **Step 4: GREEN**

- [ ] **Step 5: Commit**

### Task 8: MCP cluster tools (vendor-neutral)

**Files:**
- Modify: `src/llamamanager/mcp_server.py`
- Modify: `src/llamamanager/mcp_config.py`
- Test: `tests/test_mcp_server.py`

**Interfaces:**
- `list_nodes`
- `get_node_hardware`
- `list_node_models`
- `recommend_placement`
- `run_safe_benchmark`
- `get_cluster_status`
- dangerous/full-control: `revoke_peer`, explicit trust mutation tools.

- [ ] **Step 1: Test standard cluster reads available without Codex-specific names**

- [ ] **Step 2: Test trust mutations omitted unless full-control + dangerous `manage_cluster_trust`**

- [ ] **Step 3: RED**

- [ ] **Step 4: Implement tool wrappers over ClusterClient/Agent**

- [ ] **Step 5: GREEN and commit**

### Task 9: Cluster-aware Web and host selector

**Files:**
- Modify: `src/llamamanager/web.py`
- Modify: `src/llamamanager/templates/base.html`
- Add/Modify: cluster templates
- Modify: locale catalogs
- Test: `tests/test_web.py`
- Test: `tests/test_i18n_completeness.py`

**Interfaces:**
- global selected host context;
- `/cluster` overview;
- `/benchmarks` local/remote history view.

- [ ] **Step 1: Test host selector, selected-host rendering and EN/RU parity**

- [ ] **Step 2: RED**

- [ ] **Step 3: Implement cluster pages without duplicating domain logic**

- [ ] **Step 4: GREEN**

- [ ] **Step 5: Commit**

### Task 10: Optional mDNS discovery

**Files:**
- Create: `src/llamamanager/cluster_discovery.py`
- Modify: `src/llamamanager/cluster_server.py`
- Modify: `pyproject.toml`
- Test: `tests/test_cluster_discovery.py`

**Interfaces:**
- advertise/discover `_llamamanager._tcp.local.` candidates;
- discovery output contains only public identity/address metadata.

- [ ] **Step 1: Test discovery never creates peer trust and does not expose secret fields**

- [ ] **Step 2: RED**

- [ ] **Step 3: Implement optional zeroconf adapter**

- [ ] **Step 4: GREEN**

- [ ] **Step 5: Commit**

### Task 11: Field validation and release gate

**Files:**
- Modify: `README.md`
- Modify: `docs/CLUSTER.md`
- Modify: `docs/BENCHMARKS.md`
- Modify: `VERSION`
- Release artifacts under `/mnt/data`.

**Interfaces:** documented install/pair/inspect/placement flow plus verified ZIP/checksum.

- [ ] **Step 1: Local full-suite verification**

```bash
python -m pytest -q
python -m compileall -q src
```

- [ ] **Step 2: Package prerelease and install on two LAN nodes**

- [ ] **Step 3: Pair nodes with explicit fingerprint verification**

- [ ] **Step 4: Inspect both directions and test CPU-only/GPU placement decisions**

- [ ] **Step 5: Run safe benchmark probe and verify persistence**

- [ ] **Step 6: Document field results and only then mark the prerelease complete**

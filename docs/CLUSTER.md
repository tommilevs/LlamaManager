# Cluster / LlamaManager Brain

LlamaManager `0.3.0.dev7` adds the first deterministic multi-node control plane. A physical/virtual host running LlamaManager is a **LlamaNode**. Any node with the Web UI / MCP enabled can act as the **Brain** for nodes it explicitly trusts.

There is no permanent coordinator requirement and no local LLM requirement. Cluster state is explicit configuration plus local observations.

## Node identity

On first native installation the installer generates a stable cluster identity under:

```text
/opt/llama/llamamanager/secrets/cluster/
```

The public identity file contains:

- `node_id`: random UUID;
- `display_name`: operator-friendly host label.

The cluster directory is owned by the dedicated `llamacluster` runtime user at mode `0700`; individual credential files are mode `0600`.

The cluster fingerprint is the grouped SHA-256 digest of the stable `node_id`. It is a short human comparison aid for the current pairwise-trust model, not a CA/signing-certificate fingerprint and not a claim that a persistent signing key already exists.

## Cluster Gateway

The dedicated service is:

```text
llamamanager-cluster.service
```

It runs as the unprivileged `llamacluster` user, talks to the privileged local Agent over the Agent Unix socket, and exposes only the curated cluster API. It does **not** expose generic shell execution, generic systemd controls, or arbitrary filesystem access.

Default binding is loopback:

```text
127.0.0.1:8092
```

To expose it deliberately on a LAN:

```bash
llamarun cluster config set --name llama-4090 --bind 0.0.0.0 --port 8092
llamarun cluster config set --advertise-address http://192.168.1.20:8092
llamarun cluster enable
```

`advertise_address` is intentionally separate from `bind_host`; peers must never learn `0.0.0.0` as a route target.

## Pairing

Pairing is explicit, pairwise and non-transitive. Discovery never creates trust.

### Recommended: Join URI

On the node that should approve the relationship, create a short-lived one-time offer:

```bash
llamarun cluster join create
```

The Join URI has the form:

```text
lmjoin://v1/<base64url(canonical-json)>
```

It carries only the temporary offer material required for pairing: protocol version, inviter `node_id`, display name, human fingerprint, one or more normalized cluster Gateway endpoints, expiry, and a high-entropy single-use join ticket.

The Join URI is therefore a short-lived secret. Do not paste it into chats, issue trackers or shell history. On the joining node, omit the positional URI so `llamarun` prompts for it with hidden input:

```bash
llamarun cluster pair --local-address http://192.168.1.21:8092
```

A Join URI can carry both a LAN endpoint and an HTTPS/DNS endpoint. The peer record retains that endpoint set; future routing can prefer one endpoint and fall back safely for idempotent operations.

After successful pairing the invitation is consumed and cannot be replayed. Creating a new invitation invalidates an older pending ticket on that inviter.

For advanced/automated uses the URI can still be supplied positionally, but it may then be recorded by shell history/process tooling.

### Legacy endpoint-first flow

`0.3.0.dev3` still accepts the older endpoint-first flow for compatibility:

```bash
# Node B
llamarun cluster pair offer --address http://192.168.1.20:8092

# Node A
llamarun cluster pair http://192.168.1.20:8092
```

The newer Join URI flow is preferred because it carries identity, expiry and endpoint metadata in one single-use artifact.

The `pair accept` command remains available only for older operator workflows; normal Join URI pairing performs the authenticated handshake automatically.

## Pairing handshake

The Join URI path uses protocol **v2**.

The inviter creates an ephemeral X25519 key pair with the one-time ticket. The joiner creates its own ephemeral X25519 key pair, both sides derive a shared secret, and transcript-specific request/response proofs are computed from HKDF-SHA256 derived material that also includes the ticket-derived secret. The transcript covers both identities, both ephemeral public keys, the inviter fingerprint, both endpoint sets, and expiry metadata.

Long-lived credentials are derived directionally from the authenticated shared secret; a single reusable bearer token is not transferred between the nodes. The protocol is intentionally small and project-specific; it is not described as PAKE or Noise.

The old v1 challenge/nonce path remains for compatibility with older local tooling but is not used by the default Join URI workflow.

## Authenticated request format

After pairing, normal cluster requests use HMAC-SHA256 over:

```text
METHOD\nPATH\nTIMESTAMP\nNONCE\nSHA256(BODY)
```

Headers:

```text
X-Llama-Node: <caller node_id>
X-Llama-Credential: <credential id>
X-Llama-Timestamp: <unix seconds>
X-Llama-Nonce: <random nonce>
X-Llama-Signature: <hex hmac>
```

The receiver verifies timestamp skew and nonce replay before executing the operation.

## Peer endpoints

A trusted node identity can store several endpoints simultaneously. Typical configuration is:

```text
LAN:   http://192.168.165.33:8092        priority 10, preferred
HTTPS: https://llama-b.example.net       priority 20
```

`base_path` is supported for reverse proxies, for example:

```text
https://example.net/llama-b
```

Canonical identity matching ignores a trailing slash but preserves the path component. Query strings and fragments are rejected.

Endpoint administration is available through CLI/Agent and standard MCP tools:

```bash
llamarun cluster endpoint add NODE_ID URL --kind lan --priority 10 --preferred
llamarun cluster endpoint add NODE_ID URL --kind https --priority 20
llamarun cluster endpoint prefer NODE_ID URL
llamarun cluster endpoint remove NODE_ID URL
```

The ClusterClient sorts endpoint attempts deterministically:

1. preferred endpoint;
2. lower numeric priority;
3. stable URL ordering as a tie-breaker.

For idempotent reads, connect failures and timeouts may fall back to another configured endpoint. Authentication failures and other HTTP responses are terminal.

Mutating operations are more conservative: fallback is allowed only when the previous endpoint clearly failed before request establishment (for example DNS/connect/refused). A timeout after connection is treated as ambiguous delivery and is **not** replayed to another endpoint automatically.

## Public and protected routes

Minimal public bootstrap routes include:

- `GET /health`;
- `GET /cluster/v1/info`;
- `POST /cluster/v1/pair/challenge` (legacy v1);
- `POST /cluster/v1/pair/accept` (legacy v1);
- `POST /cluster/v2/pair/accept` (proof-gated v2 bootstrap endpoint).

Normal cluster operations require peer authentication:

- `GET /cluster/v1/host/inspect`;
- `POST /cluster/v1/placement/recommend`;
- `POST /cluster/v1/benchmark/run`;
- `GET /cluster/v1/builds/list`;
- `GET /cluster/v1/builds/status`;
- `POST /cluster/v1/runtime/start`;
- `POST /cluster/v1/runtime/stop`;
- `POST /cluster/v1/runtime/status`.

The HTTP runtime routes map only to the bounded local Agent runtime API. The Cluster Gateway does not gain generic shell, arbitrary systemd, or arbitrary path primitives.

## Brain-controlled runtime (dev6+)

The first managed runtime contract is intentionally narrow: one runtime per deterministic `build/profile` identity, for example `ik-llama-cpp:cpu`. A model must be explicit and resolve under configured model roots; arbitrary filesystem paths are rejected.

The built-in `cpu-balanced` runtime preset is the current field baseline for CPU validation:

```text
threads:     10
context:     8192
gpu_layers:  0
host:        127.0.0.1
```

Managed runtime ports are allocated automatically from:

```text
18080..18179
```

The runtime server always binds loopback in this milestone. It is a managed backend endpoint, not a new unauthenticated cluster API.

Remote control is available through the existing authenticated cluster trust path:

```bash
llamarun cluster runtime start NODE_ID \
  --build ik-llama-cpp \
  --profile cpu \
  --model Llama-3.2-3B-Instruct-Q4_K_M.gguf \
  --preset cpu-balanced

llamarun cluster runtime status NODE_ID --build ik-llama-cpp --profile cpu
llamarun cluster runtime stop NODE_ID --build ik-llama-cpp --profile cpu
```

Starting the same runtime identity twice returns a conflict. Replacement is explicit:

```bash
llamarun cluster runtime start NODE_ID ... --replace
```

`0.3.0.dev7` preserves that application-level conflict across the remote control path instead of misreporting it as peer unavailability. Runtime status also reconciles persisted state with the current systemd process start timestamp after reboot, exposing `created_at` separately from the current `process_started_at` evidence.

## Build artifact inventory

Host inspection distinguishes a Git checkout from a runnable build. Each recipe now reports bounded artifact state for known profiles:

```json
{
  "recipe_id": "ik-llama-cpp",
  "installed": true,
  "artifacts": {
    "cpu": {
      "built": true,
      "server": true,
      "cli": true,
      "bench": true,
      "build_dir": "/opt/llama/builds/ik-llama-cpp/build/cpu"
    }
  }
}
```

This fixes the earlier ambiguity where `installed: true` meant only that source existed.

## Benchmark evidence in host.inspect

The benchmark database now stores richer optional evidence needed by the Brain: node ID, build/profile/backend, threads, context, GPU layers, model bytes, measurement kind and source. Host inspection exposes a compact `benchmark_summary` so a remote Brain can reason from persisted evidence without running a new benchmark.

## Application HTTP errors across the cluster hop (dev7)

The Cluster Gateway distinguishes local Agent application responses from local Agent transport failures. If the Agent answers with a client/application status such as `400`, `404` or `409`, the Gateway preserves that status and safe error detail. Genuine Agent connection/timeout failures remain availability errors.

The calling ClusterClient likewise preserves authenticated peer application `4xx` responses instead of wrapping them as `PeerUnavailable`. This is important for runtime lifecycle semantics: an already-active runtime is a conflict, not a dead peer.

## Placement and fit

`recommend_placement()` is deterministic. The request may specify:

- model size in bytes;
- desired backend (`cpu`, `vulkan`, `cuda`, `auto`);
- preferred build recipe;
- `gpu_layers` intent;
- whether split/multi-device execution is allowed.

The first milestone intentionally uses conservative single-node fit checks with a 15% reserve. CPU-only hosts are first-class candidates for CPU placement when a CPU build profile is runnable. Mixed/unknown accelerator fit remains conservative until explicit VRAM evidence exists.

## Benchmarks

`run_probe()` may collect only cheap, non-disruptive evidence by default. See [`BENCHMARKS.md`](BENCHMARKS.md).

## mDNS discovery

When `zeroconf` is available, the cluster service may advertise `_llamamanager._tcp.local.` on the LAN. Discovery returns endpoint candidates only. In `0.3.0.dev4+`, advertisements include actual resolvable IP addresses in DNS records; clients then discard records whose `node_id` matches the local node, so a host does not present itself as a peer candidate.

Discovery never transfers secrets and never creates trust.

## MCP cluster tools

With cluster tools enabled, MCP exposes standard resource/action tools such as:

- `list_nodes`;
- `get_node_hardware`;
- `list_node_models`;
- `recommend_placement`;
- `get_cluster_status`;
- `run_safe_benchmark`;
- `list_peer_endpoints`;
- `add_peer_endpoint` / `remove_peer_endpoint` / `prefer_peer_endpoint`;
- `start_node_runtime` / `stop_node_runtime` / `get_node_runtime`;
- `create_join_offer`;
- `revoke_peer` (full-control, dangerous);
- `manage_cluster_trust` (default off).

The protocol does not depend on Codex. Any MCP client with the required capability may use it.

# 🦙 LlamaManager

LlamaManager is a Linux-first manager for multiple `llama.cpp` forks, heterogeneous CPU/GPU hosts, shared + local GGUF model libraries, per-model router presets, Hugging Face downloads, systemd services, and a small Web UI.

**Version:** 0.3.0.dev7 — **The Herd Awakens** 🦙

## What v0.1 includes

- `/opt/llama/builds` source/build registry with seven known llama.cpp variants.
- Hardware strategies `auto`, `vulkan`, `cpu`, `cuda` plus external profiles for RX 5500 XT, RTX 4090, RTX 5060 and Tesla V100 32GB.
- Multi-device discovery: a host may contain several NVIDIA/AMD GPUs plus CPU.
- Shared/local model sources with **local > shared** resolution and a safe generated `/opt/llama/models` symlink view.
- YAML model presets compiled into llama.cpp router `models-preset.ini`.
- Hugging Face GGUF search, capability hints, quant discovery, split-shard grouping, `mmproj` / MTP / draft / imatrix discovery, token auth and selective downloads.
- English + Russian i18n with English fallback.
- SQLite model index and benchmark history.
- Host Agent API + Web UI, native systemd services, and a Docker Web UI option.
- Safe migration of known legacy `/opt/llama/llama-*` trees into `/opt/llama/builds`.


## Development milestones

The `0.2.0.dev1` milestone introduced the Web/security foundation for the 0.2 series: browser-local `Auto` / `Light` / `Dark` themes, CSRF protection for Web mutations, and safe Hugging Face token writes that refuse empty values.

The `0.2.0.dev2` milestone adds an **optional MCP server** for local LLM/agent control. MCP is disabled by default, runs as the unprivileged `llamamanager` user, supports Streamable HTTP and stdio, and can use either bearer-token authentication or explicitly configured `none` mode for trusted closed networks. The default access policy is read-only; full-control mutations and dangerous operations are opt-in. A temporary Developer Mode can expose managed localization tools without permanently bloating the normal tool list.

`0.2.0.dev2+policy1` hardened live MCP policy switching so a policy-changing command waits until the running MCP registry has applied the exact new capability fingerprint.

`0.3.0.dev1` adds the first **LlamaManager Brain / LlamaNode** cluster milestone: pairwise node federation, directional HMAC credentials, host inspection, conservative model fit checks, deterministic placement recommendations, non-disruptive benchmark probes, cluster-aware Web UI, and cluster-aware standard MCP tools. The Brain is deterministic infrastructure; an LLM is optional and lives above it.

`0.3.0.dev3` adds multi-endpoint peer routing on top of secure Join URI pairing: one trusted node identity can keep LAN and HTTPS/DNS endpoints simultaneously, reverse-proxy base paths are supported, read operations can fail over safely, endpoint administration is available through CLI/Agent/MCP, and CPU-only ik_llama hosts are first-class placement targets.

`0.3.0.dev6` turns the validated two-node LAN foundation into concrete Brain-controlled runtime management. It keeps the dev4/dev5 field fixes and adds runnable build-artifact inventory, richer persisted benchmark evidence, a loopback-only `cpu-balanced` runtime preset, automatic managed ports `18080..18179`, and authenticated remote runtime start/status/stop for a model resolved from configured model roots.

`0.3.0.dev7` hardens the field-tested runtime control plane: application HTTP errors such as an already-active runtime keep their original status across Agent → Cluster Gateway → ClusterClient instead of becoming false `503` availability failures, and runtime status distinguishes the persisted runtime creation time from the current systemd process start after reboot.

## Supported build recipes

| ID | Repository | Backends |
|---|---|---|
| `llama-cpp` | ggml-org/llama.cpp | CPU, CUDA, Vulkan |
| `llama-cpp-bee` | Anbeeld/beellama.cpp | CPU, CUDA, Vulkan |
| `llama-cpp-atomic` | AtomicBot-ai/atomic-llama-cpp-turboquant | CPU, CUDA, Vulkan |
| `llama-cpp-lune` | CoderDayton/lune-turboquant | CPU, CUDA, Vulkan |
| `llama-cpp-buun` | spiritbuun/buun-llama-cpp | CPU, CUDA, Vulkan |
| `llama-cpp-turboquant` | TheTom/llama-cpp-turboquant | CPU, CUDA, Vulkan |
| `ik-llama-cpp` | ikawrakow/ik_llama.cpp | CPU, CUDA |

Forks change independently. LlamaManager validates Git state and never resets a dirty checkout automatically; a fork may still require manual adaptation after upstream changes.

## Installation prerequisites

LlamaManager requires **Python 3.11+** with the standard-library `venv` module. On Debian/Ubuntu the installer handles this automatically: if `python3` or venv support is missing, it uses `apt-get` to install `python3` / `python3-venv` and then retries. This requires root access and working APT repositories.

On distributions without `apt-get`, install Python 3.11+ and the matching venv package with the OS package manager before running `install.sh`. A global `pip` package and a global `python` command/alias are **not required**; LlamaManager always uses its private environment at `/opt/llama/llamamanager/venv`.

If a previous failed installation left an incomplete virtual environment (for example, `venv/bin/python` exists but `pip` does not), the installer detects it and recreates the venv automatically.

On upgrades, the installer reloads systemd and restarts the LlamaManager Agent and Web services, then verifies that each enabled service is active. This ensures long-running Python processes actually load the newly installed application version.

## Install on the first RX 5500 XT host

Unpack the release and run:

```bash
cd LlamaManager-v0.3.0.dev7-mvp
sudo ./scripts/install.sh --migrate --web-host 0.0.0.0
```

`--migrate` moves only the known legacy build directories such as `/opt/llama/llama-cpp` into `/opt/llama/builds/llama-cpp`. It does **not** touch model files.

If you want to inspect the migration first:

```bash
sudo ./scripts/install.sh
sudo llamarun migrate plan
```

Then perform it explicitly:

```bash
sudo llamarun migrate apply
```

The native installer creates:

```text
/opt/llama/
├── builds/
├── models/
└── llamamanager/
```

and installs:

```text
llamamanager-agent.service
llamamanager-web.service
llamamanager-mcp.service      # installed but disabled by default
llamamanager-cluster.service  # installed but disabled by default
/usr/local/bin/llamarun
```

## First checks

```bash
llamarun hardware
llamarun models sources
llamarun models scan
llamarun models rebuild
llamarun builds list
```

Web UI defaults to port `8090`. With `--web-host 0.0.0.0`, open `http://HOST_IP:8090/` on the LAN.

## Add a local model path

```bash
llamarun models source-add /data/llm_models --type local --priority 100
llamarun models rebuild
```

The default shared source created by the installer has priority `10`; local priority `100` therefore shadows the same model from SMB without deleting the SMB copy.

## Hugging Face

Store a token locally (never in shared YAML):

```bash
llamarun hf set-token hf_xxx
```

Search:

```bash
llamarun hf search "Qwen3.5 2B"
```

Inspect available quantizations and companion files:

```bash
llamarun hf inspect unsloth/Qwen3.5-4B-GGUF
```

Download one quantization selectively:

```bash
llamarun hf download unsloth/Qwen3.5-4B-GGUF \
  --quant Q4_K_M \
  --destination /data/llm_models \
  --companions mmproj,mtp
```

The Web UI also provides Hugging Face search and a details page where a quantization and individual companion files can be selected.

## MCP integration

MCP is installed but **disabled by default**. Its secure defaults are `127.0.0.1:8091`, endpoint `/mcp`, read-only access, and bearer-token authentication. Enable and inspect it with:

```bash
llamarun mcp status
llamarun mcp token regenerate
llamarun mcp enable
llamarun mcp tools
```

For a trusted closed LAN you may deliberately disable MCP authentication:

```bash
llamarun mcp config set --bind 0.0.0.0 --auth none
```

LlamaManager allows this configuration but shows a warning because every client able to reach the MCP port then receives the configured MCP capabilities. Prefer token mode whenever the network is not fully trusted.

Full-control remains separate from authentication:

```bash
llamarun mcp config set --access full-control --auth token
```

Developer localization tools are temporary and absent from the normal MCP tool list:

```bash
llamarun mcp developer enable --for 1h --group localization
llamarun mcp developer disable
```

For local process integration, stdio uses the same policy/tool registry:

```bash
llamarun mcp stdio
```

See [`docs/MCP.md`](docs/MCP.md) for the security model, tool groups, Developer Mode semantics, and client endpoint details.

## Cluster / LlamaManager Brain

The cluster gateway is installed but disabled by default on `127.0.0.1:8092`. Each installation remains a standalone LlamaNode; pairing is explicit and trust is pairwise rather than transitive.

```bash
llamarun cluster status
llamarun cluster config set --name llama-4090 --bind 0.0.0.0
llamarun cluster enable
llamarun cluster pair --address http://HOST_IP:8092
```

The Web UI adds Cluster and Benchmarks pages plus a global host selector. MCP gains vendor-neutral cluster inspection/placement tools; Codex, DeepSeek Harness, other MCP clients, and a future local Brain Agent all use the same standard schemas.

Pairing over plain HTTP is intended only for a trusted LAN. See [`docs/CLUSTER.md`](docs/CLUSTER.md) for the pairing protocol, threat model, placement semantics and benchmark safety rules.

## Tests

```bash
python -m pytest -q
```

The current source tree contains 287 automated tests.

## Development notes

- LlamaManager is intentionally Linux-first.
- Build recipes and hardware profiles are data files so forks/hardware can be added without rewriting the manager.
- Auto preset generation never overwrites a hand-written preset.
- Database failures do not destroy model files; rebuild with `llamarun models rebuild`.
- Hugging Face credentials stay local to each host.
- Never expose the Unix-socket Agent directly over the network; native Web and MCP services remain clients of the Agent boundary.

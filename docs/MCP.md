# MCP Integration

LlamaManager exposes its existing domain services over an **optional** Model Context Protocol (MCP) server. MCP does not own builds, models, Hugging Face, benchmark, or service logic; it is a thin protocol/policy layer over the same service functions used by CLI, Agent and Web.

The server targets:

```text
mcp>=2,<3
```

and supports:

- Streamable HTTP at `http://HOST:8091/mcp`;
- stdio mode for local process integration.

## Default posture

MCP is installed but **disabled by default**.

Default config:

```yaml
mcp:
  enabled: false
  bind_host: 127.0.0.1
  port: 8091
  endpoint: /mcp
  access: read-only
  auth: token
  audit_log: /opt/llama/llamamanager/logs/mcp-audit.jsonl
  dangerous:
    update_build: false
    install_build: false
    build_variant: false
    download_model: false
    restart_service: false
    manage_hf_token: false
```

Bearer-token auth is the default even on loopback.

## Threat model and privilege boundary

The native MCP HTTP service runs as the unprivileged `llamamanager` user. It does **not** run as root.

Privileged mutations go through the existing Agent Unix socket:

```text
MCP client
  -> llamamanager-mcp.service (User=llamamanager)
  -> /run/llamamanager/agent.sock
  -> llamamanager-agent.service (root)
  -> bounded LlamaManager operation
```

The MCP surface never exposes:

- arbitrary shell commands;
- arbitrary filesystem reads/writes;
- arbitrary systemd controls;
- generic privileged execution.

## Authentication modes

### `token` (default)

Enable MCP and generate a token:

```bash
llamarun mcp token regenerate
llamarun mcp enable
```

The raw token is shown only when generated. LlamaManager stores only a SHA-256 digest in:

```text
/opt/llama/llamamanager/secrets/mcp-token.sha256
```

The digest file is mode `0600`. HTTP verification uses constant-time comparison.

Clients send:

```text
Authorization: Bearer <token>
```

### `none`

Trusted closed networks may deliberately disable MCP authentication:

```bash
llamarun mcp config set --bind 0.0.0.0 --auth none
```

LlamaManager permits this but prints a warning. With `auth: none`, every client that can reach the MCP port receives the configured capability set.

Authentication is separate from authorization. `auth: none` does **not** imply full-control.

## Authorization model

Two access modes exist:

```text
read-only
full-control
```

`read-only` exposes inventory/search/status tools and read-only resources.

Mutating tools require `full-control`.

Potentially disruptive mutations additionally require their explicit `mcp.dangerous.*` flag. For example, enabling `full-control` is not enough to expose `update_build`; `dangerous.update_build` must also be true.

## Normal tool groups

Normal read-only tools include operations such as:

- `get_hardware_inventory`
- `list_builds`
- `get_build_status`
- `list_models`
- `get_model`
- `search_huggingface`
- `inspect_huggingface_repo`
- `get_benchmark_history`
- `get_service_status`
- `get_config_summary`
- `list_nodes`
- `get_node_hardware`
- `list_node_models`
- `recommend_placement`
- `get_cluster_status`
- `run_safe_benchmark`
- `list_peer_endpoints`
- `get_node_runtime`

Standard `full-control` cluster actions include endpoint management and Brain-controlled runtime lifecycle (`start_node_runtime`, `stop_node_runtime`) through the authenticated Gateway → Agent path. Pairwise trust is not implicitly editable: `create_join_offer` and `revoke_peer` remain behind the separate `manage_cluster_trust` dangerous flag.

Mutating tools such as build updates, downloads and service restart are omitted unless both access policy and dangerous flag permit them.

When a tool is not authorized, it is **omitted from the MCP tool list** rather than exposed and made to fail later.

## Resources

Read-only resources include stable text or JSON views such as:

```text
llamamanager://config/summary
llamamanager://presets/models
llamamanager://cluster/status
```

Resources that contain raw secrets are never exposed.

## Prompts

Prompts provide guidance, not privilege. Initial prompts include:

- hardware/build recommendation guidance;
- safe model-download workflow;
- benchmark interpretation guidance;
- cluster placement guidance.

A prompt never grants tools that policy removed.

## Audit log

Every MCP tool invocation appends one JSON object per line to:

```text
/opt/llama/llamamanager/logs/mcp-audit.jsonl
```

A record contains:

- timestamp;
- transport;
- authenticated client label if available;
- tool name;
- success/failure;
- duration;
- safe argument summary.

Secrets such as raw tokens and Hugging Face credentials are omitted/redacted.

## stdio

Local stdio mode uses the same registry and policy:

```bash
llamarun mcp stdio
```

It does not create a second implementation.

## CLI management

Useful commands:

```bash
llamarun mcp status
llamarun mcp config show
llamarun mcp config set --access full-control --auth token
llamarun mcp token regenerate
llamarun mcp enable
llamarun mcp disable
llamarun mcp tools
llamarun mcp resources
llamarun mcp developer enable --for 1h --group localization
llamarun mcp developer disable
```

## Docker

Docker mode packages only the Web UI in this milestone. The MCP service remains native beside the Agent.

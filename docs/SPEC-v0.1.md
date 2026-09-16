# LlamaManager v0.1 MVP Specification

## Product goal

Provide a Linux-first control plane for multiple llama.cpp forks and heterogeneous CPU/GPU machines without coupling model storage, build storage, runtime presets or Web UI state.

## Filesystem contract

```text
/opt/llama/
├── builds/
│   ├── llama-cpp/
│   ├── llama-cpp-bee/
│   ├── llama-cpp-atomic/
│   ├── llama-cpp-lune/
│   ├── llama-cpp-buun/
│   ├── llama-cpp-turboquant/
│   └── ik-llama-cpp/
├── models/        # generated symlink view only
└── llamamanager/  # config/state/db/secrets
```

Physical models remain where the operator stores them (SMB and/or local disks).

## Build abstraction

Each build recipe declares repo URL, branch policy and supported backend strategies.

Required built-in strategies:

- `auto`
- `vulkan`
- `cpu`
- `cuda`

`vulcan` is accepted as a compatibility typo alias for `vulkan`.

External hardware profiles:

- `rx5500xt`
- `rtx4090`
- `rtx5060`
- `tesla-v100-32gb`

Dirty Git repositories are never reset or updated automatically.

## Model abstraction

Model sources have type `shared` or `local` and numeric priority. Resolution rules:

1. higher priority wins;
2. equal priority: local wins over shared;
3. generated `/opt/llama/models` is rebuilt safely and only contains symlinks created by LlamaManager.

Companion roles: `mmproj`, `mtp`, `draft`, `imatrix`.

## Preset abstraction

Shared presets provide portable defaults; machine-local presets override them. Generated router INI always uses named model sections.

## Hugging Face

Requirements:

- search by Hub metadata/tags;
- identify capability hints (text, vision, audio, TTS, ASR);
- enumerate GGUF quantizations;
- group split shards;
- detect companion files;
- download selected quant only;
- secrets local only, token mode `0600`.

## i18n

English source/default locale; Russian built in. Missing translations fall back to English.

## State

YAML is source of truth for configuration. SQLite stores rebuildable model index plus benchmark history.

## Agent/Web

Native Agent is the hardware/build/model boundary. Native Web and Docker Web use the same Agent API. Native services are managed by systemd.

## Safety invariants

- no model deletion during view rebuild;
- no dirty Git reset/update;
- no secrets in shared config;
- no arbitrary privileged command API;
- no unmanaged `/opt/llama/models` file deletion outside the generated tree.

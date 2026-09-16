# Presets

Shared preset defaults can be overridden by machine-local preset files.

Example source YAML:

```yaml
models:
  Llama-3.2-3B:
    default:
      ctx_size: 8192
      gpu_layers: 99
    machines:
      llama-rx5500:
        gpu_layers: 45
```

Compile to llama.cpp router INI:

```bash
llamarun presets generate
```

Generated sections use model identifiers directly:

```ini
[Llama-3.2-3B]
ctx-size=8192
gpu-layers=45
```

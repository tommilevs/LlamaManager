# Benchmarks

LlamaManager stores benchmark results in SQLite through `StateDB.add_benchmark()`.

## Safe default policy

The first cluster iteration deliberately avoids starting inference daemons or evicting models to gather benchmark data.

Allowed default probes are lightweight and bounded:

- host capability inspection;
- existing benchmark-history lookup;
- optional disk-read probe on an already-local model file.

Forbidden by default:

- launching `llama-server` / `llama-bench`;
- loading a GGUF into CPU RAM or VRAM;
- stopping an active service;
- taking over a GPU that is already serving work.

## Persisting cluster measurements

The helper:

```python
persist_cluster_measurement(db, measurement, build_sha="...")
```

stores the same benchmark evidence used by cluster decisions in the normal benchmark table. Measurements can therefore be reused later instead of being repeated.

For `0.3.0.dev6`, persisted measurements also carry evidence needed for placement and later Brain decisions:

- `node_id`;
- `build_sha`;
- `build_profile`;
- `backend`;
- `threads`;
- `context`;
- `gpu_layers`;
- `model_bytes`;
- `measurement_kind`;
- `source`.

The schema migration is backward-compatible: older rows remain readable, and the richer columns are optional for historical results.

`StateDB.benchmark_summary()` returns a compact host-inspection view with a total count and recent benchmark evidence. This is the persistence layer for future deterministic Brain recommendations; no local LLM is required.

## Disk probe

`BenchmarkRunner.probe_model_read()` performs a small bounded read from the beginning and, for larger files, the tail of a GGUF. It reports MiB/s and bytes sampled. It does not claim to be a model throughput benchmark.

## Future benchmark types

The storage schema already supports arbitrary benchmark names and payloads. Future guarded benchmarks can add prompt-processing (`pp`) and token-generation (`tg`) metrics when the operator explicitly opts into model loading.

# Model sources

LlamaManager maintains model sources with two kinds:

- `shared`
- `local`

The resolver chooses the highest-priority source for duplicate filenames; with equal priority, `local` wins.

The generated `/opt/llama/models` tree is entirely managed by LlamaManager. `rebuild_view()` replaces only that generated tree and never deletes physical GGUF files from shared/local sources.

Sidecar/companion roles include:

- `mmproj`
- `mtp`
- `draft`
- `imatrix`

The SQLite model index is rebuildable:

```bash
llamarun models scan
llamarun models rebuild
llamarun models list --search Qwen
```

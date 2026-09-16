# Hugging Face integration

LlamaManager supports:

- search using model cards/tags;
- capability hints for text / vision / audio / TTS / ASR;
- GGUF quantization discovery;
- split-shard grouping;
- companion discovery: `mmproj`, MTP, draft, imatrix;
- selective downloads.

Token storage:

```text
/opt/llama/llamamanager/secrets/hf_token
```

with mode `0600`.

Example:

```bash
llamarun hf inspect owner/repo
llamarun hf download owner/repo --quant Q4_K_M --destination /data/llm_models --companions mmproj
```

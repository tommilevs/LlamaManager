# Configuration

The layered configuration order is:

1. built-in defaults;
2. `/etc/llamamanager/config.yaml`;
3. `/opt/llama/llamamanager/config.yaml`;
4. environment variables (currently `LLAMAMANAGER_WEB_HOST`, `LLAMAMANAGER_WEB_PORT`);
5. machine overrides from `/opt/llama/llamamanager/machine.yaml`.

Important default paths:

- builds: `/opt/llama/builds`
- generated model view: `/opt/llama/models`
- app state: `/opt/llama/llamamanager`
- database: `/opt/llama/llamamanager/state.db`
- Hugging Face token: `/opt/llama/llamamanager/secrets/hf_token`

The built-in shared model source is `/mnt/llm_models/storage/llm_models` with priority `10`.

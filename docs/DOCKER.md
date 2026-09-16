# Docker Compose

Docker mode packages only the Web UI. The privileged Agent stays native on the host and owns hardware/build/model operations.

Start the native Agent first, then:

```bash
cd docker
docker compose up --build -d
```

The compose file bind-mounts the Agent Unix socket and config directory read-only into the Web container.

This keeps native and containerized Web UIs on the same Agent API contract and avoids privileged GPU/systemd control inside the container.

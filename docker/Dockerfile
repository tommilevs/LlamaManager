FROM python:3.11-slim
WORKDIR /app
COPY pyproject.toml README.md /app/
COPY src /app/src
RUN pip install --no-cache-dir .
ENV LLAMAMANAGER_CONFIG=/config/config.yaml
ENV LLAMAMANAGER_AGENT_SOCKET=/run/llamamanager/agent.sock
CMD ["uvicorn", "llamamanager.web:app", "--host", "0.0.0.0", "--port", "8090"]

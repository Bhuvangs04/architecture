# 08: Deployment, DevOps & Infrastructure — AiTwin

---

## 1. System Requirements & Hardware Sizing

AiTwin combines real-time asynchronous Python services with local deep learning models (embeddings and LLM inference).

### Minimum Hardware Specifications (Single-Node Development / Testing):
- **CPU:** 8 physical cores (x86_64 or Apple Silicon ARM64).
- **RAM:** 32 GB system memory.
- **GPU:** NVIDIA RTX 3090 / 4090 (24 GB VRAM) or equivalent.
- **Storage:** 100 GB NVMe SSD (fast vector index and model weights read).
- **OS:** Ubuntu 22.04 LTS / Debian 12 / Windows 11 with WSL2.

### Recommended Production Specifications (Per Worker Node):
- **CPU:** AMD EPYC or Intel Xeon (16+ cores).
- **RAM:** 64 GB ECC RAM.
- **GPU:** NVIDIA A10G (24 GB) or NVIDIA A100 (40 GB / 80 GB).
- **Storage:** 500 GB NVMe SSD.
- **Network:** 10 Gbps low-latency networking.

---

## 2. Environment Variables Configuration

| Variable Name | Default Value | Description | Production Recommendation |
| :--- | :--- | :--- | :--- |
| `MONGO_URI` | `mongodb://localhost:27017/twin` | MongoDB connection string | MongoDB Atlas replica set URI with SSL |
| `MONGO_DB_NAME` | `chatdb` | Database name for chat, users, and locks | `chatdb_prod` |
| `CHROMA_DB_PATH` | `./chroma_db` | Filesystem path for persistent vector store | Fast NVMe mount (e.g., `/var/data/chroma`) |
| `UPLOAD_DIR` | `./tmp/chromadb` | Staging directory for uploaded documents | Shared persistent volume |
| `DOWNLOAD_DIR` | `./downloads` | Directory for downloadable files | S3 / Cloud Storage bucket mount |
| `MLFLOW_TRACKING_URI` | `http://localhost:5003` | URL for MLflow tracking server | Dedicated centralized MLflow instance |
| `RAG_INTERNAL_KEY` | `internal_secret_key_123` | Shared secret header for `/internal/*` | High-entropy 64-char random token |
| `ALLOWED_ORIGINS` | `http://localhost:3000,http://localhost:5173` | Allowed CORS origins for browser security | Production frontend domains |
| `RAG_CHATBOT_SERVICE_URL` | `http://5.9.97.18:8001` | Address of core service for matchmaking | Internal service DNS name |
| `USER_SERVICE_URL` | `https://www.aitwin.anivalegames.com:8943` | External user profile backend | Production User Service endpoint |
| `SSL_VERIFY` | `false` | Verify SSL certificates for HTTPX calls | `true` in production |

---

## 3. Containerized Deployment Architecture (Docker & Docker Compose)

A complete production deployment coordinates four primary services:
1. **vLLM Inference Container** (GPU accelerated)
2. **MongoDB Container** (or external MongoDB Atlas)
3. **MLflow Tracking Container**
4. **AiTwin Core Service Container**

### Production `docker-compose.yml` Reference:

```yaml
version: '3.8'

services:
  # 1. vLLM LLM Serving Engine
  vllm:
    image: vllm/vllm-openai:latest
    container_name: aitwin-vllm
    runtime: nvidia
    environment:
      - HUGGING_FACE_HUB_TOKEN=${HF_TOKEN}
    ports:
      - "8000:8000"
    volumes:
      - ~/.cache/huggingface:/root/.cache/huggingface
    command: >
      --model mistralai/Mistral-7B-Instruct-v0.3
      --port 8000
      --max-model-len 4096
      --gpu-memory-utilization 0.90
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped

  # 2. MongoDB Database
  mongodb:
    image: mongo:6.0
    container_name: aitwin-mongo
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
    restart: unless-stopped

  # 3. MLflow Observability Server
  mlflow:
    image: ghcr.io/mlflow/mlflow:v2.11.0
    container_name: aitwin-mlflow
    ports:
      - "5003:5000"
    environment:
      - BACKEND_STORE_URI=sqlite:////mlflow/mlflow.db
      - DEFAULT_ARTIFACT_ROOT=/mlflow/artifacts
    volumes:
      - mlflow_data:/mlflow
    command: mlflow server --backend-store-uri sqlite:////mlflow/mlflow.db --default-artifact-root /mlflow/artifacts --host 0.0.0.0 --port 5000
    restart: unless-stopped

  # 4. AiTwin Core Application
  aitwin-core:
    build:
      context: ./AiTwin_Model
      dockerfile: Dockerfile
    container_name: aitwin-core
    ports:
      - "8001:8001"
    environment:
      - MONGO_URI=mongodb://mongodb:27017/chatdb
      - MONGO_DB_NAME=chatdb
      - CHROMA_DB_PATH=/app/data/chroma_db
      - MLFLOW_TRACKING_URI=http://mlflow:5000
      - RAG_INTERNAL_KEY=${RAG_INTERNAL_KEY}
    volumes:
      - chroma_data:/app/data/chroma_db
      - uploads_data:/app/tmp/chromadb
    depends_on:
      - vllm
      - mongodb
      - mlflow
    restart: unless-stopped

volumes:
  mongo_data:
  mlflow_data:
  chroma_data:
  uploads_data:
```

### Production `Dockerfile` for `AiTwin_Model`:

```dockerfile
FROM python:3.11-slim

ENV PYTHONUNBUFFERED=1 \
    DEBIAN_FRONTEND=noninteractive

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    curl \
    git \
    && rm -rf /var/lib/apt/lists/*

COPY req.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r req.txt

COPY src/ ./src/
COPY mistral_tool_template.jinja ./

WORKDIR /app/src

EXPOSE 8001

HEALTHCHECK --interval=15s --timeout=5s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8001/health || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8001", "--workers", "1"]
```

---

## 4. Monitoring, Health Checks & Observability

### Health Readiness Probe:
- **Endpoint:** `GET /health`
- **Output:** `{"status": "ok", "service": "AiTwin Model Service"}`
- Kubernetes Liveness & Readiness Probes should point to `/health` with an initial delay of 30 seconds (allowing model loading).

### MLflow Tracing & Metrics:
- Every persona ingestion records `ingest_duration_ms` and `has_deep_traits`.
- Every similarity search records `candidate_count` and `matched_count`.
- Every ice-breaker generation records `generation_duration_ms` and `message_length_chars`.
- Every LLM interaction creates a parent span tagged with `user.id`, `peer.id`, and `persona.id`.

---

## 5. Production Reliability & Circuit Breaking

1. **Agent Circuit Breaker:**
   `chat_processors.py` maintains an `AgentCircuitBreaker` instance:
   - Tracks failures per agent name.
   - Threshold: 3 consecutive failures.
   - State transition: `CLOSED` $\rightarrow$ `OPEN` $\rightarrow$ `HALF_OPEN` after 60 seconds recovery timeout.
   - Protects the system from cascading timeouts if vLLM becomes overloaded.
2. **Supervisor Timeout Enforcement:**
   `AgentSupervisor.coordinate_agents()` wraps model calls with `asyncio.wait_for(timeout=30)`. Queries taking longer than 30 seconds are aborted with a clean user-facing error.
3. **MongoDB Connection Resiliency:**
   `connect_to_mongo()` uses a 5000ms server selection timeout and executes `client.admin.command('ping')` on startup to fail fast if the database is unreachable.

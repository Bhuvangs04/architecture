# 12: Setup & Local Execution Guide — AiTwin

---

## 1. Prerequisites & Environment Setup

Before launching the AiTwin platform, ensure the following dependencies are installed on your host system:

- **Operating System:** Linux (Ubuntu 22.04+ recommended) or Windows 11 with WSL2.
- **Python:** Version 3.10 or 3.11 (Python 3.12 is not recommended due to PyTorch / AutoGen compatibility constraints).
- **GPU Driver & CUDA:** NVIDIA Driver 535+ with CUDA 12.1+ (for local vLLM and BGE embeddings).
- **Database:** MongoDB 6.0+ (local daemon or MongoDB Atlas URI).
- **Node/Browser:** Any modern browser supporting WebSockets for `app_agent.html`.

---

## 2. Step-by-Step Installation

### Step 1: Clone Repository & Create Virtual Environment
```bash
# Navigate to the project directory
cd e:/Anivale/aitwin

# Create virtual environment
python -m venv venv

# Activate virtual environment
# On Linux / macOS / WSL:
source venv/bin/activate
# On Windows PowerShell:
.\venv\Scripts\Activate.ps1
```

### Step 2: Install Python Dependencies
```bash
cd AiTwin_Model
pip install --upgrade pip setuptools wheel
pip install -r req.txt
```

---

## 3. Starting Supporting Services

### Step 3: Launch Local MongoDB
```bash
# Using Docker:
docker run -d --name aitwin-mongo -p 27017:27017 -v mongo_data:/data/db mongo:6.0

# Verify connection:
mongosh --eval "db.adminCommand('ping')"
```

### Step 4: Launch MLflow Tracking Server
```bash
# Launch MLflow on port 5003
mlflow server \
  --backend-store-uri sqlite:///mlflow.db \
  --default-artifact-root ./mlflow_artifacts \
  --host 0.0.0.0 \
  --port 5003
```

### Step 5: Launch vLLM Inference Server
```bash
# Run vLLM with Mistral-7B-Instruct-v0.3
python -m vllm.entrypoints.openai.api_server \
  --model mistralai/Mistral-7B-Instruct-v0.3 \
  --port 8000 \
  --max-model-len 4096 \
  --gpu-memory-utilization 0.90

# Test vLLM health:
curl http://127.0.0.1:8000/v1/models
```
*(Alternatively, you can launch **LM Studio**, load `Mistral-7B-Instruct-v0.3`, and start the local server on port 8000).*

---

## 4. Starting the AiTwin Application

### Step 6: Set Environment Variables
```bash
# Linux / macOS / WSL
export MONGO_URI="mongodb://localhost:27017/chatdb"
export MONGO_DB_NAME="chatdb"
export CHROMA_DB_PATH="./chroma_db"
export MLFLOW_TRACKING_URI="http://localhost:5003"
export RAG_INTERNAL_KEY="internal_secret_key_123"

# Windows PowerShell
$env:MONGO_URI="mongodb://localhost:27017/chatdb"
$env:MONGO_DB_NAME="chatdb"
$env:CHROMA_DB_PATH="./chroma_db"
$env:MLFLOW_TRACKING_URI="http://localhost:5003"
$env:RAG_INTERNAL_KEY="internal_secret_key_123"
```

### Step 7: Launch the Core FastAPI Server
```bash
cd AiTwin_Model/src
python -m uvicorn main:app --host 0.0.0.0 --port 8001 --reload
```
You should see:
```
Connecting to MongoDB Atlas...
✅ Successfully connected to MongoDB Database: chatdb
MongoDB indexes ensured at startup.
AiTwin Model Service started. MLflow per-request runs enabled.
Uvicorn running on http://0.0.0.0:8001
```

### Step 8: Launch the Matchmaking Microservice (Optional)
```bash
# In a separate terminal
cd AiTwin_Model/src/matchmaking
python -m uvicorn main_1:app --host 0.0.0.0 --port 8002 --reload
```

---

## 5. Testing & Verification

### Interactive Web UI Testbed:
Open your browser and navigate to:
```
http://localhost:8001/
```
1. Enter your `User ID` (e.g., `user_alice`).
2. Select a persona (e.g., `technologist` or `dancer`).
3. Click **Start Model Chat** to chat with your own AI Twin.
4. Open a second incognito browser tab, enter `user_bob`, and click **Start Peer Chat** with `user_alice` to test real-time P2P and AI proxy switching!

### Running Automated Test Suites:
```bash
cd AiTwin_Model
pytest tests/test_grounding.py -v
```
To run the live grounding matrix (requires active vLLM):
```bash
pytest tests/test_icebreaker_grounding_matrix.py -v
```

---

## 6. Troubleshooting Common Issues

| Issue / Error | Root Cause | Solution |
| :--- | :--- | :--- |
| `Failed to connect to MongoDB: ServerSelectionTimeoutError` | MongoDB daemon is not running or port 27017 is blocked | Check `docker ps` or verify `MONGO_URI`. |
| `RuntimeError: Model loading failed` | Hugging Face cannot download `BAAI/bge-base-en-v1.5` | Check internet access or configure `HF_HOME` with pre-downloaded weights. |
| `CUDA out of memory` during vLLM startup | GPU VRAM is under 16 GB | Launch vLLM with `--gpu-memory-utilization 0.70` or use an AWQ 4-bit quantized model (`TheBloke/Mistral-7B-Instruct-v0.3-AWQ`). |
| `Failed to initialize OpenAI client for VLLM` | vLLM is not running on port 8000 | Verify vLLM is running at `http://127.0.0.1:8000/v1`. |
| `ChromaDB error: Collection already exists` | Concurrent collection creation | Handled by `get_or_create_collection()` in `rag_system.py`. Clear `./chroma_db` for a fresh start. |

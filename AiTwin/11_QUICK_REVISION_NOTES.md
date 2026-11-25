# 11: Quick Revision Notes & Cheat Sheet — AiTwin

---

## 1. Project at a Glance

- **Project Name:** AiTwin (Autonomous Digital Twin Social & Matchmaking Platform)
- **Core Value:** Keeps social and dating conversations active when users are offline via AI Proxy mode and autonomous Twin-to-Twin background social exploration.
- **Core Rule:** **Factual Grounding** — Personality is NOT Memory. An adventurous persona cannot invent trips to Tokyo.
- **Key Frameworks:** FastAPI, WebSockets, MongoDB (Motor), Microsoft AutoGen, vLLM (Mistral-7B-Instruct-v0.3), ChromaDB, Hugging Face Transformers (`BAAI/bge-base-en-v1.5`), MLflow Tracing.

---

## 2. Architecture & Concurrency Matrix

```
User A (Online)  + User B (Online)   --> Normal P2P WebSocket Chat (peer_chat_task_processor)
User A (Online)  + User B (Offline)  --> AI Proxy Mode (rag_peer_chat_processor)
User A (Offline) + User B (Offline)  --> Autonomous Twin-to-Twin Chat (aitwin_to_aitwin_processor)
```

- **Distributed Mutex:** MongoDB `locks` collection with atomic `update_one($setOnInsert)` and 300-second TTL.
- **Human Preempt:** Twin-to-Twin chat checks `chat_manager.is_connected()` before every turn; if a human logs in, the twin task is immediately aborted.
- **Rate Limiting:** Sliding window `UserRateLimiter` (20 LLM reqs / min / user).
- **Circuit Breaker:** Opens after 3 consecutive failures; 60s recovery cooldown.

---

## 3. Mathematical Formulas Used in AiTwin

### A. Memory Reranking Formula (`memory_utils.py`)
$$\text{Score} = 0.50 \cdot S_{\text{similarity}} + 0.25 \cdot S_{\text{recency}} + 0.25 \cdot S_{\text{importance}}$$
- $S_{\text{recency}}$ decays linearly from 1.0 (now) to 0.0 at 180 days.
- $S_{\text{importance}}$ ranges from 0.0 (filler) to 1.0 (critical personal facts like allergies, job, name).

### B. Matchmaking Compatibility Formula (`matchmaking/main_1.py`)
$$\text{Compatibility} = 0.35 \cdot S_{\text{semantic}} + 0.25 \cdot S_{\text{interest}} + 0.10 \cdot S_{\text{tone}} + 0.10 \cdot S_{\text{humor}} + 0.10 \cdot S_{\text{values}} + 0.10 \cdot S_{\text{activity}}$$

### C. Geospatial Proximity Filter (Haversine Formula)
$$d = 2R \arcsin\left(\sqrt{\sin^2\left(\frac{\Delta\phi}{2}\right) + \cos(\phi_1)\cos(\phi_2)\sin^2\left(\frac{\Delta\lambda}{2}\right)}\right)$$
- Hard threshold: Max radius 1.0 km (options: 50m, 500m, 1000m).

---

## 4. Database Collections & ChromaDB Topology

### MongoDB (`chatdb`):
- `users`: User preferences and `model_proxy_enabled`.
- `conversations`: Threads, `participants`, `extra_info` (AI match summary and status).
- `messages`: Polymorphic messages, `status`, `received_status`, `model_replied`.
- `locks`: Mutex locks (`lock_id`, `locked_at`, `expires_at`).

### ChromaDB (`./chroma_db`):
- `generalized_collection`: Global documents and knowledge base.
- `chat_history_collection`: Episodic messages (AI-generated messages strictly excluded!).
- `persona_definitions_collection`: Persona profiles and BGE dense embeddings.
- `peer_chat_unified_collection`: Peer-to-peer shared files.

---

## 5. Top 10 Interview Talking Points

1. **"Personality is NOT Memory":** Universal grounding rule in `factual_grounding.md`. Prevents LLMs from translating traits into historical experiences.
2. **AI Memory Poisoning Shield:** Messages with `model_replied=True` are blocked from RAG vector storage, stopping hallucinated claims from becoming permanent memories.
3. **Dual Storage Design:** MongoDB for flexible, polymorphic transactional documents + ChromaDB for local vector similarity.
4. **Autonomous Twin Pacing:** Max 6 turns, 60s cap, 0.4s inter-turn pacing, evaluated via `summarize_and_evaluate_chat()`.
5. **Two-Stage Validation:** Generation followed by `SystemPromptEnforcer.validate_factual_grounding()`. On failure, retries with explicit anti-hallucination prompt.
6. **Local vLLM Inference:** Private self-hosted Mistral-7B eliminates per-token API costs and keeps intimate social conversations on private GPUs.
7. **BGE-Base Dense Embeddings:** 768-dim normalized embeddings running locally via Hugging Face Transformers.
8. **MemoryFilter Intelligence:** Rejects messages shorter than 8 chars, filler patterns, and emoji spam before vector indexing.
9. **Zero-Invention Rule:** If information is unknown, the twin defers naturally instead of hallucinating.
10. **Preemptible Distributed Concurrency:** Atomic MongoDB locking with TTL guarantees single-worker execution with instant human takeover.

# 13: Architecture Verification & Codebase Evidence — AiTwin

---

## 1. Inventory & Inspection Summary

An exhaustive reverse engineering analysis was conducted across the AiTwin repository. Below is the verified source inventory:

| Category | File / Path | Lines of Code | Key Classes / Functions Identified |
| :--- | :--- | :--- | :--- |
| **API Entrypoint** | `AiTwin_Model/src/main.py` | 744 | `startup_event`, `websocket_peer_chat_endpoint`, `ingest_persona`, `vector_similarity_search_endpoint` |
| **Agent Core** | `AiTwin_Model/src/agents.py` | 1,811 | `RAGRouterAgent`, `SystemPromptEnforcer`, `UserRateLimiter`, `generate_ice_breaker_direct_sync` |
| **Vector Engine** | `AiTwin_Model/src/rag_system.py` | 967 | `RAGSystem`, `embed_texts_cached`, `ingest_persona`, `ingest_messages_for_rag`, `rag_function_call` |
| **State Machine** | `AiTwin_Model/src/peerChatManager.py` | 448 | `PeerChatManager`, `connect`, `disconnect`, `try_start_aitwin`, `load_history_from_cache` |
| **Chat Workers** | `AiTwin_Model/src/chat_processors.py` | 823 | `peer_chat_task_processor`, `rag_peer_chat_processor`, `message_hold_processor`, `AgentCircuitBreaker` |
| **Autonomous Twin** | `AiTwin_Model/src/aitwin_chat_processor.py` | 112 | `aitwin_to_aitwin_processor`, turn pacing, summary extraction |
| **WebSocket Utils** | `AiTwin_Model/src/chat_utils.py` | 57 | `websocket_listener_task`, input size clamping |
| **Memory Layer** | `AiTwin_Model/src/memory_utils.py` | 514 | `MemoryFilter`, `MemoryClassifier`, `ImportanceScorer`, `MemoryReranker`, `MemoryCache` |
| **Persona Config** | `AiTwin_Model/src/persona_manager.py` | 54 | `ALL_PERSONAS`, `get_persona_data` |
| **Database Pool** | `AiTwin_Model/src/mongodb.py` | 46 | `connect_to_mongo`, `close_mongo_connection`, `get_mongo_db` |
| **Data Access** | `AiTwin_Model/src/mongo_crud.py` | 318 | `create_message`, `acquire_aitwin_lock`, `release_aitwin_lock`, `get_messages_for_conversation` |
| **Document Schemas** | `AiTwin_Model/src/mongo_models.py` | 85 | `User`, `Conversation`, `Message`, `AITwinLock`, `MessageType`, `MessageStatus` |
| **Observability** | `AiTwin_Model/src/mlflow_logger.py` | 176 | `MLflowLogger`, `start_span`, `log_latency`, `record_error` |
| **Sentiment RL** | `AiTwin_Model/src/matchllmreinforcementsentiment.py` | 158 | `ConversationalReinforcementModel`, VADER polarity scoring |
| **Common Utils** | `AiTwin_Model/src/utils.py` | 48 | `create_text_message`, `load_llm_config`, `_record_metrics` |
| **Frontend Testbed** | `AiTwin_Model/src/app_agent.html` | 845 | WebSocket UI, persona picker, read receipt tracker |
| **Matchmaking** | `AiTwin_Model/src/matchmaking/main_1.py` | 814 | `POST /matches/find`, `POST /matches/initiate`, Haversine formula, scoring formula |
| **Grounding Policy**| `AiTwin_Model/src/config/factual_grounding.md`| 676 | 17 Universal Anti-Hallucination rules, 5-tier evidence hierarchy |
| **Guardrails YAML** | `AiTwin_Model/src/config/guardrails.yaml` | 48 | Regex intent patterns and response contracts |
| **Model Config** | `AiTwin_Model/src/config/model_config.yaml` | 16 | vLLM endpoint configuration and token limits |
| **Prompts YAML** | `AiTwin_Model/src/config/prompts.yaml` | 40 | Rephrase and primary prompt templates |
| **Unit Tests** | `AiTwin_Model/tests/test_grounding.py` | 105 | Validator rejection test, RAG AI-message blocking test |
| **Integration Tests**| `AiTwin_Model/tests/test_icebreaker_grounding_matrix.py` | 117 | Live validation test matrix A through G |
| **Jinja Template** | `AiTwin_Model/mistral_tool_template.jinja` | 1 | Mistral v0.3 tool formatting template |

---

## 2. Directly Verified Facts vs Inferred Rationale

### Directly Verified Facts:
1. **Adversarial RAG Shield:** Verified in `rag_system.py` lines 592–660. Messages with `model_replied == True` are skipped during vector ingestion.
2. **Distributed Mutex Lock:** Verified in `mongo_crud.py` lines 50–96. The lock uses `update_one` with `$setOnInsert` and a 300-second expiration.
3. **Factual Grounding Rules:** Verified in `config/factual_grounding.md` and enforced via `SystemPromptEnforcer.validate_factual_grounding()` in `agents.py`.
4. **Autonomous Twin Execution:** Verified in `aitwin_chat_processor.py`. Alternates up to 6 turns between two offline users and cancels immediately if a human reconnects.
5. **Multi-Factor Matchmaking Scoring:** Verified in `matchmaking/main_1.py` lines 87–150. Weights: Semantic 35%, Interest 25%, Tone 10%, Humor 10%, Values 10%, Activity 10%.

### Inferred Rationales (Clearly Labeled):
1. **Selection of Embedded ChromaDB over Qdrant:** Inferred that embedded ChromaDB was chosen to keep the development setup completely self-contained on a single developer workstation without needing external Docker container orchestration.
2. **PostgreSQL Dependency in `req.txt`:** `psycopg2-binary` appears in `req.txt`, but no PostgreSQL connection code exists in `src/`. Inferred that PostgreSQL was either used in a legacy version or intended for a future relational migration.
3. **Internal Key Check Commented Out:** In `main.py` lines 144–148, the internal key validation is currently commented out (`return True`). Inferred that this was bypassed during staging/testing between the matchmaking microservice and the core service.

---

## 3. Unverified Components & Identified Architectural Action Items

1. **Unused Dependencies in `req.txt`:**
   - `psycopg2-binary` (no active PostgreSQL calls).
   - `faiss-cpu` (ChromaDB handles active vector search; FAISS is unused in `src/`).
   - `opentelemetry-sdk` / `opentelemetry-instrumentation-fastapi` (MLflow tracing handles observability; native OTel exporter is unconfigured).
   *Recommendation:* Remove unused packages to reduce Docker container build time.
2. **Internal Key Authentication Enforcement:**
   In `main.py`, uncomment the internal key check before deploying to a public production network:
   ```python
   def verify_internal_key(internal_key: str = Header(None)):
       if internal_key != os.environ.get("RAG_INTERNAL_KEY", "internal_secret_key_123"):
           raise HTTPException(status_code=401, detail="Invalid internal service key")
       return True
   ```
3. **Hardcoded Server IP in HTML & Microservice:**
   `app_agent.html` and `matchmaking/main_1.py` contain references to `http://5.9.97.18:8001`. Ensure these are parameterized via environment variables (`RAG_CHATBOT_SERVICE_URL`) in container configurations.

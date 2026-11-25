# 07: Codebase Deep Dive & Source Inspection — AiTwin

---

## 1. Directory Structure & Module Responsibilities

The codebase resides in `AiTwin_Model/src/`. Below is a comprehensive breakdown of every major source file, its purpose, its key classes and functions, and its execution mechanics.

```
AiTwin_Model/src/
|-- main.py                          # Application entry point, HTTP & WebSocket endpoints
|-- agents.py                        # AutoGen agent implementations & grounding enforcer
|-- rag_system.py                    # ChromaDB vector store, BGE embeddings, RAG pipeline
|-- peerChatManager.py               # WebSocket session & connection lifecycle manager
|-- chat_processors.py               # Async workers for P2P, AI Proxy, and Message Hold
|-- aitwin_chat_processor.py         # Autonomous background Twin-to-Twin execution engine
|-- chat_utils.py                    # WebSocket listener task and message validation
|-- memory_utils.py                  # Memory intelligence layer (Filter, Classifier, Reranker)
|-- persona_manager.py               # Static persona definitions & trait getters
|-- mongodb.py                       # Motor async client connection pool
|-- mongo_crud.py                    # Async MongoDB queries, mutations, and distributed locks
|-- mongo_models.py                  # Pydantic schemas for database documents
|-- mlflow_logger.py                 # Structured logging & MLflow distributed tracing wrapper
|-- matchllmreinforcementsentiment.py # VADER sentiment analyzer & conversational reinforcement
|-- utils.py                         # Message constructors & YAML loaders
|-- app_agent.html                   # Interactive browser UI for testing & demo
|-- config/
|   |-- factual_grounding.md         # 17 universal anti-hallucination rules
|   |-- guardrails.yaml              # Regex rules for intent and response contracts
|   |-- model_config.yaml            # vLLM / Mistral-7B endpoint configuration
|   |-- prompts.yaml                 # Rephrase and primary prompt templates
|-- matchmaking/
    |-- main_1.py                    # Standalone Matchmaking & Social microservice
```

---

## 2. Core Modules Inspection

### A. `main.py` (FastAPI Core Server)
- **Role:** Central gateway, route registration, lifecycle management.
- **Key Functions & Handlers:**
  - `startup_event()`: Connects to MongoDB via `connect_to_mongo()`; creates background indexes on `messages`, `users`, and `conversations`.
  - `shutdown_event()`: Closes MongoDB client connections cleanly.
  - `websocket_peer_chat_endpoint(websocket, user_id, peer_id, persona_id)`: The core WebSocket route. Manages the dynamic transition between P2P chat, AI proxy chat, and autonomous twin chat.
  - `websocket_model_chat_endpoint(websocket, user_id, persona_id)`: Direct interaction between a human user and their AI Twin.
  - `ingest_persona(persona_data)`: REST endpoint converting `PersonaIngest` schemas into ChromaDB embeddings and persistent records.
  - `vector_similarity_search_endpoint(payload)`: Computes cosine similarity of candidate persona embeddings for matchmaking.
  - `generate_icebreaker_by_id_endpoint(request)`: Resolves personas by user ID, generates a grounded ice-breaker, and persists it into MongoDB.

---

### B. `agents.py` (Agent Architecture & Factual Grounding)
- **Role:** Agent definitions, prompt construction, guardrail enforcement, and LLM communication.
- **Key Classes & Functions:**
  - `UserRateLimiter`: Thread-safe sliding window rate limiter (Lock-guarded, default: 20 requests/minute/user).
  - `RAGRouterAgent(AssistantAgent)`: Custom AutoGen agent. Injects `config/factual_grounding.md`, builds persona prompts, enforces first-person perspective, and registers `retrieve_rag_context` tool.
  - `SystemPromptEnforcer`: Validates generated responses against context using `validate_factual_grounding()`. If the LLM generates claims unsupported by context, the enforcer returns `UNSUPPORTED`.
  - `_generate_validated_response(user_prompt)`: Orchestrates the generation $\rightarrow$ validation $\rightarrow$ retry $\rightarrow$ fallback loop. If retry fails, falls back to safe deflection: *"I'm not exactly sure how to answer that, but I'd love to learn more!"*
  - `generate_ice_breaker_direct_sync(p_a, p_b)`: Direct synchronous LLM call bypassing AutoGen overhead to construct persona-grounded ice-breaker messages.
  - `enforce_first_person(text)`: Post-processing regex replacing third-person references (e.g., "Sam likes...") with first-person ("I like...").
  - `apply_response_contract(text, is_proxy)`: Truncates responses to a maximum of 3 sentences and strips greeting cliches when proxying.

---

### C. `rag_system.py` (Vector Search & Knowledge Retrieval)
- **Role:** ChromaDB management, dense embeddings, chunking, and context retrieval.
- **Key Classes & Methods:**
  - `RAGSystem.__init__()`: Loads `BAAI/bge-base-en-v1.5` tokenizer and model; initializes ChromaDB client and collections.
  - `embed_texts_cached(texts)`: Generates normalized 768-dimensional float vectors, caching results in memory.
  - `ingest_persona(persona_id, user_id, persona_data)`: Flattens persona attributes (about_me, values, quirks, deep_traits), embeds, and upserts into `persona_definitions_collection`.
  - `ingest_messages_for_rag(messages, user_id, peer_id)`: Ingests chat messages into vector store.
    - **CRITICAL SECURITY CHECK:** Messages where `model_replied == True` are discarded to prevent AI hallucinations from poisoning permanent memory!
    - Calls `MemoryFilter.should_store()` to reject filler/greetings.
    - Calls `ImportanceScorer.score()` to calculate weight.
  - `rag_function_call(query, user_id, peer_id)`: Executes vector retrieval across collections. Queries results, passes them to `MemoryReranker.rerank()`, checks query cache, and returns formatted context.

---

### D. `peerChatManager.py` (Session & State Management)
- **Role:** Tracks connected clients, in-memory message queues, and manages AI Twin triggering.
- **Key Methods:**
  - `connect(websocket, user_id)`: Registers socket in `active_connections` and allocates an `asyncio.Queue()` in `message_queues`.
  - `disconnect(user_id, websocket)`: Deregisters socket and queue. Inspects active conversations to check if peers are offline, triggering AI Twin mode if applicable.
  - `try_start_aitwin(u1, u2)`: Calls `crud.acquire_aitwin_lock()` to ensure only one worker runs autonomous chat.
  - `clear_aitwin_running(u1, u2)`: Releases the distributed lock in MongoDB.
  - `load_history_from_cache(u1, u2)`: Retrieves recent messages from MongoDB, converting them into standard dictionary structures via `db_message_to_dict()`.

---

### E. `chat_processors.py` (Asynchronous Chat Processors)
- **Role:** Dedicated background async loops for each conversational state.
- **Key Processors:**
  - `peer_chat_task_processor`: Runs when both users are online. Forwards messages, updates delivery status (`delivered`), and persists to MongoDB.
  - `rag_peer_chat_processor`: Runs when the peer is offline and proxy is enabled. Retrieves RAG context, calls `RAGRouterAgent`, validates grounding, persists message with `model_replied=True`, and sends to the online user.
  - `message_hold_processor`: Runs when the peer is offline and proxy is disabled. Queues messages in MongoDB with status `sent`.
  - `AgentCircuitBreaker`: Implements circuit breaker pattern (`CLOSED`, `OPEN`, `HALF_OPEN`). Opens after 3 consecutive failures, blocking calls for 60 seconds.
  - `AgentSupervisor`: Wraps agent execution with `asyncio.wait_for(timeout=30)` to prevent runaway inference tasks.

---

### F. `aitwin_chat_processor.py` (Autonomous Twin-to-Twin Engine)
- **Role:** Executes background conversations when both users are offline.
- **Key Function:** `aitwin_to_aitwin_processor(user_a, user_b, chat_manager, rag_system, persona_id, max_turns=6, max_duration=60)`:
  1. Loads personas for User A and User B.
  2. Alternates speaking turns between User A and User B.
  3. Checks `chat_manager.is_connected()` before every turn. If either human connects, terminates immediately!
  4. Stores every turn in MongoDB (`model_replied=True`, `is_proxy=True`).
  5. Upon turn completion, calls `summarize_and_evaluate_chat()`.
  6. Updates MongoDB conversation `extra_info` with `ai_match_summary` and `ai_match_status` (`compatible` or `failed`).
  7. Adds a system message to chat history informing users of the match summary.
  8. Clears the distributed lock in `finally` block.

---

### G. `memory_utils.py` (Memory Intelligence Layer)
- **Role:** Eliminates noise, categorizes data, scores importance, and reranks vector results.
- **Key Classes:**
  - `MemoryFilter`: Rejects messages matching 12 regex filler patterns (`r"^(hi|ok|lol|thanks|bye)..."`) or having fewer than 8 characters.
  - `MemoryClassifier`: Categorizes memories into `project`, `preference`, `skill`, `relationship`, `identity`, or `conversation` using ordered regex heuristics.
  - `ImportanceScorer`: Assigns weights from 0.0 to 1.0 (e.g., allergies/names = 0.9–0.95, salary/job = 0.8, weather/lunch = 0.25).
  - `MemoryReranker`: Blends similarity, recency (180-day decay), and importance:
    $$\text{FinalScore} = 0.50 \cdot \text{Sim} + 0.25 \cdot \text{Recency} + 0.25 \cdot \text{Importance}$$
  - `MemoryCache`: In-memory LRU caches for query results, embeddings, and personas.

---

### H. `matchmaking/main_1.py` (Matchmaking Microservice)
- **Role:** Standalone service for finding compatible users and dispatching ice-breakers.
- **Key Logic & Endpoints:**
  - `POST /matches/find`:
    1. Fetches candidate profiles from `USER_SERVICE_URL`.
    2. Computes Haversine distance, dropping candidates beyond `matching_radius_km` (max 1.0 km).
    3. Calls `/vector_similarity_search` on the core service for persona cosine similarity.
    4. Computes Jaccard interest overlap, tone bucket similarity, humor scale distance, and values overlap.
    5. Returns ranked candidates with `score_breakdown`.
  - `POST /matches/initiate`: Executes matching, calls `/internal/generate_icebreaker_by_id`, and dispatches the opening friend request.

---

### I. `config/factual_grounding.md` (Universal Anti-Hallucination Rules)
- **Role:** Ground truth specification referenced by `RAGRouterAgent` and `SystemPromptEnforcer`.
- **Core Principles:**
  - **Evidence Hierarchy:** Level 1 (Current User Info) > Level 2 (Verified RAG Memory) > Level 3 (Explicit Profile) > Level 4 (Personality Traits) > Level 5 (Model Inference).
  - **Personality is NOT Memory:** A trait (e.g., "spontaneous", "adventurous") grants permission to speak casually or express openness, but NEVER permits claiming past experiences (e.g., claiming to have visited Spain or gone skydiving).
  - **Evidence Scope Preservation:** Evidence cannot be amplified (e.g., "interested in" must not become "passionate about" or "loves").

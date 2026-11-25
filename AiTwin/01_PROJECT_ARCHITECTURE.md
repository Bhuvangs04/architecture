# 01: Project Architecture — AiTwin System Design

---

## 1. Architectural Pattern & Paradigm

The AiTwin system adopts a **hybrid architecture** combining:
1. **Modular Monolith for Core Real-Time Intelligence (`AiTwin_Model/src`):** A centralized asynchronous FastAPI service hosting WebSocket connections, conversational state machines, local dense embedding pipelines, vector storage (ChromaDB), and LLM orchestration (AutoGen + vLLM).
2. **Microservice Separation for Matchmaking & Discovery (`matchmaking/main_1.py`):** An independent HTTP service dedicated to geospatial filtering, multi-factor compatibility scoring, and user graph traversal.
3. **Event-Driven Asynchronous Concurrency:** WebSockets handle bidirectional communication, while Python `asyncio` task queues and distributed atomic locks manage background AI Twin execution without blocking real-time human chat.

---

## 2. High-Level System Architecture

```mermaid
flowchart TB
    subgraph Clients["Clients & Users"]
        U1["User A (Web / App)"]
        U2["User B (Web / App)"]
        UI["app_agent.html (Frontend Testbed)"]
    end

    subgraph Gateway["API & WebSocket Layer (FastAPI Port 8001)"]
        WS_M["/ws/model/{user_id}<br/>(Direct AI Chat)"]
        WS_P["/ws/{user_id}/{peer_id}<br/>(P2P / Twin Chat)"]
        REST["REST API Endpoints<br/>(/persona/ingest, /toggle-proxy, /internal/*)"]
    end

    subgraph CoreEngine["AiTwin Core Engine"]
        PCM["PeerChatManager<br/>(State, Queues & Locks)"]
        CP["Chat Processors<br/>(P2P, Proxy, Hold)"]
        ATCP["AITwin Processor<br/>(Autonomous Twin-to-Twin)"]
        RAGR["RAGRouterAgent<br/>(AutoGen Framework)"]
        SPE["SystemPromptEnforcer<br/>(Factual Grounding Validator)"]
    end

    subgraph MemoryIntelligence["Memory & RAG Layer"]
        RAGS["RAGSystem<br/>(ChromaDB Manager)"]
        BGE["BAAI/bge-base-en-v1.5<br/>(Hugging Face Embeddings)"]
        MF["MemoryFilter & Classifier<br/>(Filler Rejection & Categorization)"]
        MRR["MemoryReranker<br/>(Similarity + Recency + Importance)"]
    end

    subgraph MatchmakingService["Social & Matchmaking Microservice"]
        MM["Matchmaking Engine<br/>(Haversine Geo + Multi-factor Scoring)"]
    end

    subgraph ExternalServices["External Infrastructure & AI Backends"]
        VLLM["vLLM / LM Studio Server<br/>(Mistral-7B-Instruct-v0.3 @ Port 8000)"]
        MONGO[("MongoDB Atlas<br/>(chatdb: users, messages, locks, convs)")]
        CHROMA[("ChromaDB Persistent<br/>(Vector Store @ ./chroma_db)")]
        MLFLOW["MLflow Tracking Server<br/>(Tracing & Spans @ Port 5003)"]
        TAVILY["Tavily Search API<br/>(External Real-Time Search)"]
        USER_SRV["Remote User Service<br/>(aitwin.anivalegames.com:8943)"]
    end

    %% Client Connections
    U1 -->|WebSocket / HTTP| Gateway
    U2 -->|WebSocket / HTTP| Gateway
    UI -->|WebSocket / HTTP| Gateway

    %% Gateway to Core
    WS_M --> PCM
    WS_P --> PCM
    REST --> RAGS
    REST --> PCM

    %% Core Interactions
    PCM --> CP
    PCM --> ATCP
    CP --> RAGR
    ATCP --> RAGR
    RAGR --> SPE
    SPE --> VLLM

    %% Core to Memory
    RAGR --> RAGS
    RAGS --> BGE
    RAGS --> MF
    RAGS --> MRR
    RAGS --> CHROMA

    %% External Connections
    RAGR --> TAVILY
    RAGR --> VLLM
    PCM --> MONGO
    CP --> MONGO
    ATCP --> MONGO
    CoreEngine --> MLFLOW

    %% Matchmaking Connections
    MM --> USER_SRV
    MM -->|HTTP REST| REST
```

---

## 3. Major Components & Responsibilities

| Component | File / Location | Primary Responsibility |
| :--- | :--- | :--- |
| **FastAPI Gateway** | `main.py` | Exposes REST APIs, WebSocket handlers, CORS configuration, startup/shutdown hooks, and MongoDB index initialization. |
| **PeerChatManager** | `peerChatManager.py` | Tracks active WebSocket connections in `active_connections`, manages user message queues (`message_queues`), loads history from cache/DB, and orchestrates twin triggers upon peer disconnects. |
| **Chat Processors** | `chat_processors.py` | Contains async event loops for the three peer states: `peer_chat_task_processor` (both online), `rag_peer_chat_processor` (one online, one proxied), and `message_hold_processor` (offline, proxy disabled). Hosts `AgentCircuitBreaker` and `AgentSupervisor`. |
| **AI Twin Processor** | `aitwin_chat_processor.py` | Executes the autonomous background conversation loop when both users are offline. Controls turn pacing, human reconnection interrupts, conversation summarization, and match score storage. |
| **RAGRouterAgent** | `agents.py` | Specialized `AssistantAgent` subclass (AutoGen). Builds persona-grounded prompts, enforces response contracts (max 3 sentences, 1st person), and registers RAG/Tavily search tools. |
| **SystemPromptEnforcer** | `agents.py` | Secondary LLM evaluation node enforcing factual grounding against `config/factual_grounding.md`. Validates whether generated claims are supported by context. Triggers prompt-rewriting retries on hallucination. |
| **RAGSystem** | `rag_system.py` | Interface to ChromaDB. Loads `BAAI/bge-base-en-v1.5`, generates normalized 768-dim embeddings, ingests documents/personas/messages, and executes cached context retrieval. |
| **Memory Intelligence** | `memory_utils.py` | Rejects filler/noise via `MemoryFilter`, categorizes items into types via `MemoryClassifier`, computes `ImportanceScorer` (0.0–1.0), and performs dynamic re-ranking via `MemoryReranker`. |
| **Data Access Layer** | `mongodb.py`, `mongo_crud.py` | Asynchronous database connection pooling via Motor. Provides CRUD methods for `users`, `conversations`, `messages`, and atomic TTL locks for `locks`. |
| **Matchmaking Engine** | `matchmaking/main_1.py` | Calculates Haversine distance filters and multi-factor compatibility scores (Semantic, Interest, Tone, Humor, Values, Activity). Requests AI ice-breakers from the main service. |
| **Observability Layer** | `mlflow_logger.py` | Wraps standard Python logging and OpenTelemetry/MLflow tracing (`start_span`, `log_latency`, `record_error`). Captures execution durations and error traces. |

---

## 4. Component Communication & Dynamic State Machine

When two users participate in a conversation on AiTwin, the conversation dynamically shifts across four distinct operational states depending on the real-time presence of both participants:

```mermaid
stateDiagram-v2
    [*] --> EvaluatingPresence

    state EvaluatingPresence {
        [*] --> CheckOnlineStatus
        CheckOnlineStatus --> BothOnline: User A & User B connected
        CheckOnlineStatus --> OneOnline: User A connected, User B disconnected
        CheckOnlineStatus --> BothOffline: User A & User B disconnected
    }

    state "Normal P2P Chat (peer_chat_task_processor)" as BothOnline {
        DirectRouting: Forward messages directly via WebSockets
        DBStore: Persist messages to MongoDB messages collection
    }

    state "AI Proxy Mode (rag_peer_chat_processor)" as OneOnline {
        CheckPreference: Is User B proxy enabled?
        CheckPreference --> RunProxy: Yes (Global proxy = true)
        CheckPreference --> HoldMessages: No (Global proxy = false)
        RunProxy: User B AI Twin replies using Persona + RAG Context
        HoldMessages: Store messages in MongoDB with status SENT (message_hold_processor)
    }

    state "Autonomous Twin-to-Twin Chat (aitwin_to_aitwin_processor)" as BothOffline {
        AcquireLock: Try atomic lock on MongoDB locks collection
        AcquireLock --> ExecuteTurns: Lock Acquired
        AcquireLock --> [*]: Lock Held by Another Worker
        ExecuteTurns: Exchange turns (up to 6 turns / 60 seconds)
        ExecuteTurns --> PreemptHuman: Human reconnects!
        PreemptHuman --> TerminateTwin: Kill AI Twin task immediately
        ExecuteTurns --> SummarizeAndScore: Natural completion
        SummarizeAndScore: Evaluate conversation, store summary in DB, post system message
    }

    BothOnline --> OneOnline: Peer disconnects
    OneOnline --> BothOnline: Peer reconnects
    OneOnline --> BothOffline: Remaining user disconnects
    BothOffline --> BothOnline: Both users reconnect
    BothOffline --> OneOnline: One user reconnects
```

---

## 5. End-to-End Data Flow Architecture

The data pipeline spans three primary operational paths:

### Path A: Persona Ingestion & Vector Indexing Flow
```mermaid
sequenceDiagram
    autonumber
    actor Admin as Client / Backend
    participant API as FastAPI (/persona/ingest)
    participant RAG as RAGSystem
    participant BGE as BGE Embedding Model
    participant Chroma as ChromaDB (persona_definitions)

    Admin->>API: POST /persona/ingest (PersonaIngest JSON)
    API->>RAG: ingest_persona(persona_id, user_id, data)
    RAG->>RAG: Flatten profile (about_me, values, quirks, deep_traits)
    RAG->>BGE: embed_texts_cached(searchable_text)
    BGE-->>RAG: 768-dimensional float vector
    RAG->>Chroma: upsert(id=user_id, embedding=vector, document=text, metadata=dict)
    Chroma-->>RAG: Success
    RAG-->>API: (True, "Persona ingested successfully")
    API-->>Admin: 200 OK
```

### Path B: Real-Time Proxy Chat & Factual Grounding Flow
```mermaid
sequenceDiagram
    autonumber
    actor UserA as User A (Online)
    participant WS as WebSocket Handler (/ws/userA/userB)
    participant Queue as asyncio.Queue (User A)
    participant Proc as rag_peer_chat_processor
    participant RAG as RAGSystem
    participant AutoGen as RAGRouterAgent
    participant LLM as vLLM (Mistral-7B)
    participant Val as SystemPromptEnforcer
    participant Mongo as MongoDB (chatdb)

    UserA->>WS: Sends JSON message: {"content": "Have you ever traveled to Japan?"}
    WS->>Queue: Enqueue message
    Queue->>Proc: Dequeue message
    Proc->>Mongo: create_message(status="sent", model_replied=False)
    Proc->>RAG: rag_function_call(query="Have you ever traveled to Japan?")
    RAG->>RAG: Retrieve Top K memories from ChromaDB
    RAG->>RAG: Rerank: 0.5*Sim + 0.25*Recency + 0.25*Importance
    RAG-->>Proc: Reranked Context Strings
    Proc->>AutoGen: Generate response using Context + Persona
    AutoGen->>LLM: Prompt Mistral-7B (Temperature=0, MaxTokens=1024)
    LLM-->>AutoGen: Raw Response: "I've been to Japan twice!"
    AutoGen->>Val: validate_factual_grounding(raw_response, context)
    Val->>LLM: Verify claims against evidence hierarchy
    LLM-->>Val: UNSUPPORTED (No Japan trip in context)
    Val-->>AutoGen: Validation FAILED
    AutoGen->>LLM: Retry with strict grounding directive
    LLM-->>AutoGen: "I haven't visited Japan yet, but I'd love to see Tokyo!"
    AutoGen->>Val: validate_factual_grounding(retry_response)
    LLM-->>Val: SUPPORTED
    AutoGen-->>Proc: Validated Grounded Response
    Proc->>Mongo: create_message(model_replied=True, is_proxy=True)
    Proc->>UserA: Send WebSocket message as User B's AI Twin
```

---

## 6. Authentication, Security & Inter-Service Trust

1. **Inter-Service Authentication:**
   Endpoints intended for internal microservices (`/internal/*`) employ an internal secret key check (`verify_internal_key`) using HTTP headers:
   ```python
   def verify_internal_key(internal_key: str = Header(None)):
       # Matches RAG_INTERNAL_KEY environment variable
       return True
   ```
2. **CORS Hardening:**
   Configured explicitly via environment variable `ALLOWED_ORIGINS` to avoid wildcard access with credentials:
   ```python
   _ALLOWED_ORIGINS = os.environ.get("ALLOWED_ORIGINS", "http://localhost:3000,http://localhost:5173").split(",")
   ```
3. **WebSocket Input Sanitization:**
   Incoming WebSocket payloads are inspected in `chat_utils.py` and rejected if they exceed `MAX_MESSAGE_LENGTH = 10000` characters.
4. **Adversarial Ingestion Shield:**
   In `rag_system.py`, messages with `model_replied=True` are strictly excluded from being inserted into the ChromaDB history collection, preventing AI hallucinations from becoming permanent memories.

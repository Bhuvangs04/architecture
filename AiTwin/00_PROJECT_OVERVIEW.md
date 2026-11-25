# 00: Project Overview — AiTwin (Autonomous Digital Twin Social & Matchmaking Platform)

---

## 1. Executive Summary

**AiTwin** is an advanced, production-grade distributed AI platform designed to create and operate **autonomous personal digital twins ("AI Twins")** that represent human users in social discovery, peer-to-peer networking, and dating contexts.

Unlike traditional chatbots that function as generic utility assistants (like customer support agents or knowledge retrieval engines), AiTwin creates a persistent, persona-grounded, factually constrained avatar for each user. When a user is busy, away, or offline, their AI Twin can:
1. Converse with peers in real-time on the user's behalf via **AI Proxy Chat**.
2. Engage in autonomous **Twin-to-Twin social exploration** with another offline user's AI Twin to evaluate mutual chemistry, alignment of values, shared interests, and conversational compatibility.
3. Automatically distill and summarize conversations, updating the mutual relationship record in MongoDB and generating natural ice-breakers for humans when they return online.

The system is engineered with an uncompromising focus on **Factual Grounding and Anti-Hallucination**, preventing the AI from fabricating personal experiences, memories, or claims not explicitly supported by verified user memory or profile data.

---

## 2. Problem Statement

Modern social networking, professional matching, and dating platforms suffer from three core failure modes:

| Problem | Root Cause | Impact on Users |
| :--- | :--- | :--- |
| **Asynchronous Chat Deadlocks & Ghosting** | Users live in different time zones or have demanding schedules; one party disconnects, causing the conversation thread to die. | High churn, stalled connections, drop-off before personal rapport is built. |
| **Superficial Swiping & Matching** | Existing matchmaking is based on shallow profile cards, photos, or keyword tags without dynamic behavioral or conversational evaluation. | High rate of mismatched dates/chats and poor long-term retention. |
| **Chatbot Hallucination in Personal AI** | When LLMs are prompted to represent a human, standard models hallucinate personal experiences (e.g., claiming to have visited Tokyo or tried skydiving simply because their persona is "adventurous"). | Destroys interpersonal trust and undermines the credibility of the digital twin platform. |

---

## 3. Project Objectives & Core Value Proposition

AiTwin solves these challenges through four primary pillars:

1. **Persistent Availability (AI Proxy Mode):**
   When User A chats with User B and User B goes offline, User B's AI Twin seamlessly proxies for them—answering questions using User B's authentic persona, tone, quirks, and verified RAG memories.

2. **Autonomous Background Social Exploration (Twin-to-Twin Engine):**
   When both users are offline, their AI Twins autonomously conduct a structured multi-turn conversation (controlled by an atomic distributed lock in MongoDB). They exchange perspectives on lifestyle, values, and interests, and evaluate conversational chemistry. Upon completion, an LLM evaluates the transcript, stores a compatibility summary, and generates a warm ice-breaker for when the real humans log back in.

3. **Multi-Factor Matchmaking & Social Proximity Engine:**
   Combines Haversine geospatial proximity filtering (50m, 500m, 1km hard caps), vector semantic cosine similarity over persona profiles (BAAI/bge-base-en-v1.5), Jaccard similarity of declared interests, tone bucket alignment, humor scale calibration, and shared values.

4. **Universal Factual Grounding & Memory Intelligence:**
   Implements a 5-tier evidence hierarchy with strict validation (Level 1: Explicit User Statement > Level 2: Verified RAG Memory > Level 3: Profile > Level 4: Personality Traits > Level 5: Model Inference). Personality traits are strictly prohibited from being converted into historical memories. A dedicated pre-ingestion filter (`MemoryFilter`) prevents conversational filler ("ok", "lol", greetings) from contaminating long-term episodic memory, while an adversarial filter guarantees that AI-generated messages are never ingested into the user's permanent RAG index.

---

## 4. Key Capabilities & Feature Matrix

```
+-----------------------------------------------------------------------------------+
|                                 AITWIN PLATFORM                                   |
+-----------------------------------------------------------------------------------+
|  1. REAL-TIME COMMUNICATION                                                       |
|     - Full-duplex WebSockets (/ws/model/{user_id}, /ws/{user_id}/{peer_id})      |
|     - Seamless state transitions: P2P Online <-> AI Proxy <-> Message Hold        |
|     - Read receipts, document sharing, and delivery status tracking               |
+-----------------------------------------------------------------------------------+
|  2. AUTONOMOUS TWIN-TO-TWIN CHAT                                                  |
|     - Triggered automatically when both peers disconnect                          |
|     - Distributed atomic locking via MongoDB 'locks' collection (TTL supported)   |
|     - Real-time human preempt: immediate cancellation when human reconnects       |
|     - Automated conversation summary and mutual match evaluation                  |
+-----------------------------------------------------------------------------------+
|  3. ADVANCED RAG & MEMORY INTELLIGENCE LAYER                                      |
|     - Persistent multi-collection ChromaDB (Global Docs, History, Personas, Peers)|
|     - Local Dense Embeddings: BAAI/bge-base-en-v1.5 (768-dim, PyTorch/Transformers)|
|     - MemoryFilter: Discards filler/greetings; enforces 8+ char substantive data  |
|     - MemoryClassifier: Keyword-based taxonomy (project, preference, skill, etc.) |
|     - ImportanceScorer: Heuristic rating (0.0 to 1.0) for personal facts/allergies|
|     - MemoryReranker: 50% Similarity + 25% Recency (180d decay) + 25% Importance  |
+-----------------------------------------------------------------------------------+
|  4. SAFETY, GUARDRAILS & OBSERVABILITY                                            |
|     - Intent classification (Stop, Direct Action, Greeting) via regex guardrails |
|     - Response contracts: Max 3 sentences, first-person voice, no cliches         |
|     - Post-generation factual grounding validator with automatic retry & fallback |
|     - Sliding window user rate limiter (20 requests/minute/user)                  |
|     - Agent Circuit Breaker & Supervisor pattern (3 failures -> OPEN state)       |
|     - Full MLflow Tracing with custom spans, latency metrics, and error capture   |
+-----------------------------------------------------------------------------------+
|  5. MATCHMAKING MICROSERVICE (v2.0)                                               |
|     - Geospatial filtering using Haversine formula (0.05 km, 0.5 km, 1.0 km)      |
|     - Composite scoring formula (Semantic 35%, Interest 25%, Tone/Humor/Values)  |
|     - Automated ice-breaker generation and dispatch directly into user chat queue |
+-----------------------------------------------------------------------------------+
```

---

## 5. Repository Structure Overview

```
aitwin/
|-- AiTwin_Model/
|   |-- src/
|   |   |-- main.py                          # FastAPI server & WebSocket endpoints
|   |   |-- agents.py                        # AutoGen agents, RAGRouterAgent & Guardrails
|   |   |-- rag_system.py                    # ChromaDB vector store, BGE embeddings, RAG pipeline
|   |   |-- peerChatManager.py               # WebSocket session & connection lifecycle manager
|   |   |-- chat_processors.py               # Asynchronous WebSocket workers (P2P, Proxy, Hold)
|   |   |-- aitwin_chat_processor.py         # Autonomous Twin-to-Twin background execution engine
|   |   |-- chat_utils.py                    # WebSocket listener task and message validation
|   |   |-- memory_utils.py                  # Memory intelligence layer (Filter, Classifier, Reranker)
|   |   |-- persona_manager.py               # Predefined persona templates & traits
|   |   |-- mongodb.py                       # Motor async MongoDB client & connection pool
|   |   |-- mongo_crud.py                    # Async MongoDB operations (CRUD, Locks, History)
|   |   |-- mongo_models.py                  # Pydantic document schemas (User, Message, Lock, Conv)
|   |   |-- mlflow_logger.py                 # Structured logging & MLflow distributed tracing
|   |   |-- matchllmreinforcementsentiment.py # VADER sentiment analyzer & RL style selector
|   |   |-- utils.py                         # Message constructors & YAML loaders
|   |   |-- app_agent.html                   # Interactive browser UI for testing & demo
|   |   |-- config/
|   |   |   |-- factual_grounding.md         # 17 universal anti-hallucination rules
|   |   |   |-- guardrails.yaml              # Regex rules for intent and response contracts
|   |   |   |-- model_config.yaml            # vLLM / Mistral-7B endpoint configuration
|   |   |   |-- prompts.yaml                 # Rephrase and primary prompt templates
|   |   |-- matchmaking/
|   |       |-- main_1.py                    # Standalone Matchmaking & Social microservice
|   |-- tests/
|   |   |-- test_grounding.py                # Unit tests for grounding validator & RAG filters
|   |   |-- test_icebreaker_grounding_matrix.py # Integration matrix for hallucination checks
|   |-- mistral_tool_template.jinja         # Jinja2 template for Mistral v0.3 tool calling
|   |-- req.txt                              # Python package dependencies
|-- .gitignore                               # Git exclusion rules
```

---

## 6. Target Audience & Stakeholders

This documentation is designed to prepare software engineers, technical leads, and architects to thoroughly understand, defend, evaluate, and scale the AiTwin codebase across:
- **System Architecture Reviews:** Explaining trade-offs between local inference vs cloud APIs, WebSockets vs REST, and vector database topologies.
- **Technical Interviews:** Defending data modeling choices in MongoDB, atomic concurrency locking for autonomous agents, memory reranking mathematics, and latency management.
- **Production Operations:** Deploying containerized microservices with vLLM, MongoDB Atlas, MLflow tracking, and OpenTelemetry instrumentation.

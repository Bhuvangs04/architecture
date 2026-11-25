# 04: API Documentation — AiTwin Endpoints Reference

---

## 1. Overview & Authentication

AiTwin provides both **External REST Endpoints**, **Internal Microservice Endpoints**, and **Bidirectional WebSocket Endpoints**.

- **Base URL (Core Service):** `http://5.9.97.18:8001` (or `http://localhost:8001`)
- **Base URL (Matchmaking Service):** `http://localhost:8002` (configurable)
- **WebSocket Protocol:** `ws://` / `wss://`
- **Internal Security:** Inter-service requests to `/internal/*` include the header:
  `internal-key: <RAG_INTERNAL_KEY>` (Default: `internal_secret_key_123`).

---

## 2. Core Service REST Endpoints (`main.py`)

### 1. Root & Web UI Testbed
- **Method:** `GET`
- **Path:** `/`
- **Purpose:** Serves the interactive single-page testing client `app_agent.html`.
- **Response:** `200 OK` (`text/html`).

---

### 2. Health Readiness Probe
- **Method:** `GET`
- **Path:** `/health`
- **Purpose:** Used by load balancers and orchestrators to check service availability.
- **Response Example:**
  ```json
  {
    "status": "ok",
    "service": "AiTwin Model Service"
  }
  ```

---

### 3. Active Connection Counter
- **Method:** `GET`
- **Path:** `/connections`
- **Purpose:** Returns the total count of active WebSocket connections tracked in memory.
- **Response Example:**
  ```json
  {
    "active_connections": 4
  }
  ```

---

### 4. Persona Ingestion
- **Method:** `POST`
- **Path:** `/persona/ingest`
- **Purpose:** Ingests and indexes a user's persona into ChromaDB (`persona_definitions_collection`) and computes its BGE dense embedding.
- **Request Body:**
  ```json
  {
    "persona_id": "technologist",
    "user_id": "usr_94821a",
    "name": "Alex Rivera",
    "about_me": "Senior AI engineer passionate about open source and cooking.",
    "tone": "casual",
    "values": ["curiosity", "transparency", "continuous learning"],
    "quirks": ["Uses cooking analogies for software architecture", "Coffee enthusiast"],
    "behavioral_settings": {
      "response_length": "short",
      "style": "friendly",
      "humor_level": "moderate"
    },
    "deep_traits": {
      "communication": "Direct and visual",
      "social_battery": "Ambivert"
    }
  }
  ```
- **Response:** `200 OK`
  ```json
  {
    "message": "Persona for user usr_94821a ingested into ChromaDB."
  }
  ```
- **Error Codes:** `400 Bad Request` (missing persona_id), `500 Internal Server Error`.

---

### 5. Toggle AI Proxy Preference
- **Method:** `POST`
- **Path:** `/toggle-proxy/{user_id}`
- **Purpose:** Enables or disables the global AI proxy feature for a given user.
- **Request Body:**
  ```json
  {
    "enable": true
  }
  ```
- **Response:** `200 OK`
  ```json
  {
    "message": "AI proxy preference for user 'usr_94821a' set to enabled.",
    "enabled": true
  }
  ```

---

### 6. Vector Similarity Search
- **Method:** `POST`
- **Path:** `/vector_similarity_search`
- **Purpose:** Computes cosine similarity between a source user's persona vector and a list of candidate user IDs.
- **Request Body:**
  ```json
  {
    "user_id": "usr_94821a",
    "candidate_ids": ["usr_84712b", "usr_10293c", "usr_39281d"]
  }
  ```
- **Response:** `200 OK`
  ```json
  [
    { "user_id": "usr_84712b", "score": 0.884 },
    { "user_id": "usr_39281d", "score": 0.742 },
    { "user_id": "usr_10293c", "score": 0.615 }
  ]
  ```

---

### 7. Internal: Retrieve User Persona
- **Method:** `GET`
- **Path:** `/internal/persona/{user_id}`
- **Headers:** `internal-key: <KEY>`
- **Purpose:** Fetches the full persona profile dictionary from ChromaDB for matchmaking.
- **Response:** `200 OK`
  ```json
  {
    "persona_id": "technologist",
    "user_id": "usr_94821a",
    "name": "Alex Rivera",
    "about_me": "Senior AI engineer passionate about open source and cooking.",
    "tone": "casual",
    "values": ["curiosity", "transparency"],
    "quirks": ["Uses cooking analogies for software architecture"]
  }
  ```
- **Error Codes:** `404 Not Found` if user persona is unindexed.

---

### 8. Document Upload & Ingestion
- **Method:** `POST`
- **Path:** `/upload_doc/{user_id}/{peer_id}`
- **Content-Type:** `multipart/form-data`
- **Payload:** `file: UploadFile`
- **Purpose:** Saves shared document (PDF/text), parses via PyMuPDF/fitz, chunks text, generates BGE embeddings, and stores in `peer_chat_unified_collection`. Dispatches notification to peer.
- **Response:** `200 OK`
  ```json
  {
    "message": "File 'project_notes.pdf' processed and 12 chunks indexed for RAG."
  }
  ```

---

### 9. Document Download
- **Method:** `GET`
- **Path:** `/download_doc/{filename}`
- **Purpose:** Downloads previously uploaded documents.
- **Response:** `FileResponse` with MIME `application/octet-stream`.

---

### 10. Internal: Generate Ice-Breaker by Personas
- **Method:** `POST`
- **Path:** `/internal/generate_icebreaker`
- **Headers:** `internal-key: <KEY>`
- **Request Body:**
  ```json
  {
    "sender_persona": {
      "user_id": "usr_A",
      "name": "Sam",
      "tone": "casual",
      "about_me": "I enjoy trail running and sourdough baking.",
      "values": ["outdoors", "creativity"]
    },
    "receiver_persona": {
      "user_id": "usr_B",
      "name": "Jordan",
      "tone": "friendly",
      "about_me": "Marathon runner and artisan bread enthusiast.",
      "values": ["fitness", "craft"]
    }
  }
  ```
- **Response:** `200 OK`
  ```json
  {
    "message": "Hey Jordan, noticed you're into marathon running! Have you ever baked sourdough for pre-run fuel?"
  }
  ```

---

### 11. Internal: Generate Ice-Breaker by User IDs
- **Method:** `POST`
- **Path:** `/internal/generate_icebreaker_by_id`
- **Headers:** `internal-key: <KEY>`
- **Purpose:** Resolves both personas from ChromaDB, prompts Mistral-7B, verifies grounding, creates conversation in MongoDB, and stores the initial message.
- **Request Body:**
  ```json
  {
    "sender_id": "usr_A",
    "receiver_id": "usr_B"
  }
  ```
- **Response:** `200 OK`
  ```json
  {
    "message": "Hey Jordan, noticed you're into marathon running! Have you ever baked sourdough for pre-run fuel?"
  }
  ```

---

### 12. Internal: Send Initial Friend Request / Match Message
- **Method:** `POST`
- **Path:** `/internal/friend-request/send`
- **Headers:** `internal-key: <KEY>`
- **Request Body:**
  ```json
  {
    "sender_id": "usr_A",
    "receiver_id": "usr_B",
    "message": "Hey Jordan! Would love to connect over running.",
    "match_score": 0.892
  }
  ```
- **Response:** `201 Created`
  ```json
  {
    "message": "Initial match message stored successfully.",
    "request_status": "SENT"
  }
  ```

---

### 13. Conversation Summary & Match Evaluation
- **Method:** `GET`
- **Path:** `/conversation/{user_id}/{peer_id}/summary`
- **Purpose:** Returns the autonomous twin-to-twin chat evaluation summary and compatibility status from MongoDB conversation `extra_info`.
- **Response:** `200 OK`
  ```json
  {
    "user_id": "usr_A",
    "peer_id": "usr_B",
    "conversation_id": "conv_usr_A_usr_B",
    "ai_match_summary": "Both twins discussed outdoor adventures and passion for cooking. Shared strong alignment on work-life balance.",
    "ai_match_status": "COMPATIBLE",
    "evaluated_at": "2026-11-24T18:30:15.123456"
  }
  ```

---

## 3. WebSocket API Endpoints

### A. Direct Model Chat Endpoint
- **URL:** `ws://5.9.97.18:8001/ws/model/{user_id}?persona_id=default`
- **Message Format (Client to Server):**
  ```json
  {
    "content": "Tell me about your background in machine learning.",
    "source": "usr_A",
    "status": "sent"
  }
  ```
- **Message Format (Server to Client):**
  ```json
  {
    "id": "f83a1b2c-...",
    "source": "model",
    "content": "I've spent the last 10 years working on distributed AI and GPU computing.",
    "timestamp": 1795538400.12,
    "status": "sent",
    "model_replied": true
  }
  ```

### B. Peer-to-Peer & AI Proxy Endpoint
- **URL:** `ws://5.9.97.18:8001/ws/{user_id}/{peer_id}?persona_id=default`
- **Handshake Sequence:**
  1. Connection accepted.
  2. Server immediately sends historical messages:
     ```json
     {
       "type": "history",
       "content": [ ...list of prior messages... ]
     }
     ```
  3. Real-time message exchange begins.

---

## 4. Matchmaking Microservice Endpoints (`matchmaking/main_1.py`)

### 1. Find Top Matches
- **Method:** `POST`
- **Path:** `/matches/find`
- **Request Body:**
  ```json
  {
    "user_id": "692ebcd9f23865791ac68a8d",
    "candidate_ids": ["692ebcd9f23865791ac68a8e", "692ebcd9f23865791ac68a8f"],
    "top_k": 3
  }
  ```
- **Response:** `200 OK`
  ```json
  [
    {
      "user_id": "692ebcd9f23865791ac68a8e",
      "username": "Jordan",
      "compatibility_score": 0.842,
      "score_breakdown": {
        "semantic": 0.88,
        "interest": 0.75,
        "tone": 1.0,
        "humor": 0.8,
        "values": 0.67,
        "activity": 1.0
      }
    }
  ]
  ```

### 2. Initiate Match & Dispatch Ice-Breaker
- **Method:** `POST`
- **Path:** `/matches/initiate`
- **Purpose:** Computes score, fetches ice-breaker from AI Model service, and stores the friend request in chat DB.

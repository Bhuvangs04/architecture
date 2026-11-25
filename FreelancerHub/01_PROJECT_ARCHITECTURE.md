# 01: Project Architecture & System Design

> **Document ID**: `01_PROJECT_ARCHITECTURE`  
> **System Name**: FreelancerHub  
> **Architectural Pattern**: Distributed Microservices with Edge Proxy & AI Gateway Pattern  

---

## 1. Architectural Pattern & Paradigm

The FreelancerHub system implements a **Distributed Multi-Service / Hybrid Microservices Architecture**. Rather than deploying a monolithic codebase, the project separates concerns into discrete, independently deployable services optimized for distinct operational profiles:

```
┌────────────────────────────────────────────────────────────────────────┐
│                      DISTRIBUTED TOPOLOGY OVERVIEW                     │
└────────────────────────────────────────────────────────────────────────┘
  [Client Web App]        [Admin Portal]          [AI Gateway Client]
      (Port 8080)             (Port 8081)             (Port 8080)
           │                       │                       │
           ▼                       ▼                       ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │         Edge Reverse Proxy & Custom Load Balancer (:4000)        │
  │     - Sticky Session Management (AES-256-CBC Encrypted Cookie)   │
  │     - Least-Connections Routing Algorithm                        │
  │     - IP Rate Limiting (100 req/min) & Permanent In-Memory Block │
  │     - Active Health Check Loop (every 10 seconds)                │
  └──────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                 Core Node.js / Express Backend (:5000)           │
  │     - RESTful Business Logic (Projects, Contracts, Escrow)       │
  │     - Real-Time WebSocket Server (Chat & Events)                 │
  │     - Two-Tier Role-Based Access Control (RBAC)                  │
  │     - MongoDB Persistence Layer (Mongoose ODM)                   │
  └──────────────────┬─────────────────────────────┬─────────────────┘
                     │                             │
                     ▼                             ▼
        ┌─────────────────────────┐   ┌──────────────────────────────┐
        │  MongoDB Database       │   │ Python FastAPI ML Service    │
        │  (Port 27017)           │   │ (Port 5001)                  │
        │  - Document Collections │   │ - TLAM Multiplicative Engine │
        │  - Financial Ledgers    │   │ - Success Predictor          │
        │  - Audit Logs           │   │ - Fake Profile Classifier    │
        └─────────────────────────┘   └──────────────────────────────┘
```

### Architectural Classification
1. **Frontend / Admin Presentation Tier**: Decoupled Single-Page Applications (SPAs) built with React 18, Vite, TypeScript, and Tailwind CSS.
2. **Edge Network Tier**: Dedicated custom Node.js reverse proxy (`load-balancer/`) providing traffic distribution, connection pooling, and denial-of-service defense.
3. **Core Application Tier**: Express-based REST API and WebSocket daemon coordinating business transactions, escrow funding, contract signing, and chat streaming.
4. **Machine Learning / AI Tier**: Independent Python FastAPI microservice (`freelancer-ml-project/`) exposing high-speed model inference, vector scoring, and trust algorithms.
5. **Data Tier**: MongoDB document database supporting complex nested schemas (Milestones, Deliverables, Disputes, Violations).

---

## 2. Major Components & Responsibilities

| Component | Repository Path | Tech Stack | Port | Primary Responsibility |
|---|---|---|---|---|
| **Marketplace Frontend** | `frontend/` | React 18, Vite, TS, Tailwind, Redux, TanStack Query | 8080 / 5173 | Client & Freelancer user interface; project posting, bidding, digital agreement signing, milestone tracking, Web Crypto chat. |
| **Admin Control Plane** | `Freelancer-Admin/` | React 18, Vite, TS, Radix UI, TanStack Query | 8081 | Governance dashboard; dispute arbitration, wallet freezes, strike issuances, AI key quotas, platform fee adjustments. |
| **Edge Load Balancer** | `load-balancer/` | Node.js, Express, `http-proxy`, Helmet, Crypto | 4000 | Reverse proxy; least-connections traffic forwarding, AES-256 encrypted sticky session cookies, IP rate limiting. |
| **ML Inference Service** | `freelancer-ml-project/` | Python 3.10+, FastAPI, Scikit-learn, Pandas, SBERT | 5001 | ML predictions; TLAM matchmaking, project success probability scoring, behavioral fake profile classification. |
| **Core Backend Server** | *External Microservice* | Node.js, Express, WebSocket (`ws`), Mongoose | 5000 | Core REST endpoints (`/api/vi`), user auth sessions, Razorpay payment verification, WebSocket chat broadcast. |
| **Database Server** | *Infrastructure* | MongoDB | 27017 | Persistent document store for Users, Projects, Agreements, Milestones, Escrow, Disputes, and Violations. |

---

## 3. High-Level Architecture Diagrams

### 3.1 Distributed System Architecture

```mermaid
graph TB
    subgraph Users Tier
        ClientUser["Client Browser<br/>(React SPA :8080)"]
        FreelancerUser["Freelancer Browser<br/>(React SPA :8080)"]
        AdminUser["Platform Admin<br/>(React Admin :8081)"]
    end

    subgraph Edge Layer
        LB["Load Balancer & Reverse Proxy :4000<br/>- Least Connections<br/>- AES-256 Sticky Session<br/>- Rate Limiting (100 req/min)"]
    end

    subgraph Application & Business Logic
        BackendAPI["Node.js / Express API Gateway :5000<br/>- REST Controllers (/api/vi)<br/>- WebSocket Server (/chat/:id)<br/>- Two-Tier RBAC Guard<br/>- Razorpay Webhook Receiver"]
    end

    subgraph Artificial Intelligence Tier
        MLService["FastAPI ML Service :5001<br/>- TLAM Multiplicative Engine<br/>- Success Predictor (Random Forest)<br/>- Fake Profile Detector (Gradient Boost)<br/>- SBERT Semantic Matching"]
    end

    subgraph Data & Persistence Tier
        DB[("MongoDB :27017<br/>- Users & Auth<br/>- Projects & Bids<br/>- Agreements & Escrow<br/>- Disputes & Violations")]
    end

    subgraph External Platforms
        Razorpay["Razorpay Payment Gateway<br/>(Orders, Payments, Webhooks)"]
        GitHub["GitHub REST API<br/>(Repos, Commits, Languages)"]
    end

    ClientUser -->|HTTPS & WSS| LB
    FreelancerUser -->|HTTPS & WSS| LB
    AdminUser -->|Direct HTTPS| BackendAPI
    
    LB -->|Proxy Web Traffic| BackendAPI
    BackendAPI <-->|Mongoose ODM| DB
    BackendAPI <-->|HTTP POST JSON| MLService
    BackendAPI <-->|Order Creation & Verification| Razorpay
    BackendAPI <-->|Developer Verification| GitHub
```

---

### 3.2 Edge Load Balancer Internal Architecture

The custom load balancer in [load-balancer/loadBalancer.js](file:///e:/DEVOPS/load-balancer/loadBalancer.js) acts as the intelligent ingress controller:

```mermaid
flowchart TD
    Req["Incoming HTTP/WebSocket Request"] --> Helmet["Helmet Security Middleware<br/>(Security Headers, Disable X-Powered-By)"]
    Helmet --> CORS["CORS Validation<br/>(Allowed Origins: :8080, :8081)"]
    CORS --> IPBlockCheck{"IP in blockedIPs Set?"}
    
    IPBlockCheck -- Yes --> Block403["Reject: 403 Forbidden<br/>(IP Blocked Due to Excessive Requests)"]
    IPBlockCheck -- No --> RateLimiter{"Requests > 100 in 60s Window?"}
    
    RateLimiter -- Yes --> AddBlocked["Add IP to blockedIPs<br/>Reject: 403 Forbidden"]
    RateLimiter -- No --> CookieCheck{"Has 'Server_Id' Cookie?"}
    
    CookieCheck -- Yes --> DecryptCookie["AES-256-CBC Decrypt Cookie<br/>Extract Target Backend URL"]
    DecryptCookie --> TargetActive{"Is Target Server Active?"}
    
    TargetActive -- Yes --> ForwardTarget["Forward to Sticky Server"]
    TargetActive -- No --> LeastConn["Calculate Least Loaded Server<br/>(min connections from active pool)"]
    
    CookieCheck -- No --> LeastConn
    LeastConn --> EncryptCookie["AES-256-CBC Encrypt Server URL<br/>Set-Cookie: Server_Id (HttpOnly, Secure)"]
    EncryptCookie --> ProxyForward["http-proxy: Forward Request"]
    ForwardTarget --> ProxyForward
    
    ProxyForward --> TargetBackend["Target Backend Server (:3000 / :5000)"]
    
    subgraph Health Check Loop
        HC["healthCheck() every 10,000ms"] --> Probe["GET /health on all servers"]
        Probe --> UpdateStatus["server.active = (response.status === 200)"]
    end
```

---

### 3.3 Machine Learning Microservice Architecture

The ML service in [freelancer-ml-project/api_gateway_integration.py](file:///e:/DEVOPS/freelancer-ml-project/api_gateway_integration.py) processes requests using serialized scikit-learn models and the TLAM engine:

```mermaid
flowchart TD
    MLReq["POST /api/ml/predict or /api/ml/match"] --> EndpointRouter{"Target Model / Task"}
    
    subgraph Success Prediction Pipeline
        EndpointRouter -- "success_prediction" --> SP_Params["Extract 8 Parameters:<br/>experience, projects, rating, completion_rate,<br/>on_time_rate, skill_match, completeness, budget_ratio"]
        SP_Params --> SP_Scale["StandardScaler.transform()"]
        SP_Scale --> SP_Infer["Predictor Model: RandomForest / GradientBoost"]
        SP_Infer --> SP_Proba["model.predict_proba()[:, 1]"]
        SP_Proba --> SP_Output["Return: success_probability (0-100%),<br/>recommendation, risk_level"]
    end

    subgraph Fake Profile Detection Pipeline
        EndpointRouter -- "fake_profile_detection" --> FP_Params["Extract 9 Behavioral Signals:<br/>completeness, total_skills, rating, total_reviews,<br/>projects, account_age, portfolio_items, budget_ratio, certs"]
        FP_Params --> FP_Eng["Feature Engineering:<br/>Review/Project Ratio, Skill Density, Anomaly Flags"]
        FP_Eng --> FP_Scale["StandardScaler.transform()"]
        FP_Scale --> FP_Infer["Classifier Model: GradientBoost"]
        FP_Infer --> FP_Out["Return: is_fake (bool), fake_probability (%),<br/>risk_level, red_flags list"]
    end

    subgraph TLAM Matching Engine
        EndpointRouter -- "/api/ml/match" --> TLAM_Parse["Parse Project Requirements & Candidate Pool"]
        TLAM_Parse --> TLAM_Features["Compute 6 Sub-Scores per Freelancer:<br/>1. skill_score (Jaccard Overlap)<br/>2. semantic_score (SBERT Cosine Sim)<br/>3. context_score (Domain Specialization)<br/>4. maturity_score (Log-Scaled Experience)<br/>5. behavior_score (Success Rate * Interaction)<br/>6. budget_score (Gaussian Rate Alignment)"]
        
        TLAM_Features --> FormulaChoice{"Scoring Method"}
        FormulaChoice -- "TLAM (Multiplicative)" --> TLAM_Calc["Score = (Capability^0.349) * (Reliability^0.148) * (Alignment^0.034)<br/>*Note: Any 0 yields final score 0 (Strict Matching)*"]
        FormulaChoice -- "Weighted Sum (Linear)" --> WS_Calc["Score = sum(weight_i * feature_i)<br/>*Partial Credit for Incomplete Matches*"]
        
        TLAM_Calc --> SortRank["Sort Candidates by Primary Score (Desc)"]
        WS_Calc --> SortRank
        SortRank --> Enrich["Append Model Explainer & Feature Breakdown"]
        Enrich --> TLAM_Output["Return Ranked Match JSON to AI Gateway"]
    end
```

---

## 4. Communication & Protocol Matrix

The platform orchestrates 4 distinct communication mechanisms across services:

```mermaid
sequenceDiagram
    autonumber
    participant UI as Client Browser (React SPA)
    participant LB as Edge Load Balancer (:4000)
    participant Core as Core Backend API (:5000)
    participant ML as FastAPI ML Microservice (:5001)
    participant DB as MongoDB (:27017)
    participant RZ as Razorpay Gateway

    Note over UI,RZ: Phase 1: Client Posts Project & Bids Received
    UI->>LB: POST /api/vi/client/projects
    LB->>Core: Forward Request (Sticky Session)
    Core->>DB: Insert Project Document
    Core-->>UI: Project Created

    Note over UI,ML: Phase 2: AI Matching & Success Prediction
    UI->>LB: POST /api/vi/ml/match-freelancers
    LB->>Core: Forward Request
    Core->>DB: Fetch Project Skills & Active Freelancers
    Core->>ML: POST /api/ml/match (Candidates + Project)
    ML-->>Core: Ranked Candidate Array + TLAM Explanations
    Core-->>UI: Candidate Matches with Fit Badges

    Note over UI,RZ: Phase 3: Contract Acceptance & Escrow Pre-Funding
    UI->>LB: POST /api/vi/agreement/create (Bid Accepted)
    LB->>Core: Forward Contract Creation
    Core->>DB: Insert Agreement (Status: Draft)
    UI->>LB: POST /api/vi/payments/create-order
    LB->>Core: Forward Order Request
    Core->>RZ: Create Razorpay Order (Escrow Amount)
    RZ-->>Core: Order ID & Signature Hash
    Core-->>UI: Razorpay Checkout Payload
    UI->>RZ: Direct Client Modal Payment
    RZ-->>UI: Payment Success (PaymentId, Signature)
    UI->>LB: POST /api/vi/payments/verify-payment
    LB->>Core: Verify HMAC SHA256 Signature
    Core->>DB: Update Escrow (Status: Funded) & Agreement (Status: Active)
    Core-->>UI: Payment Confirmed & Contract Activated
```

---

## 5. Authentication & Authorization Architecture

FreelancerHub implements a **Two-Tier Authentication & RBAC Defense Model**:

```mermaid
flowchart TD
    RouteReq["User Navigates to Protected Route (e.g., /freelancer/profile)"] --> Tier1{"Tier 1: Redux Client Guard<br/>(FreelancerAuth.tsx)"}
    
    Tier1 -- "isAuthenticated === false" --> RedirectSignIn["Redirect to /sign-in"]
    Tier1 -- "isAuthenticated === true" --> Tier2{"Tier 2: Server Permission Check<br/>(Pages.access.tsx via useCheckPermission)"}
    
    Tier2 --> ServerCall["POST /security/checkAuth/permission/:role<br/>(Credentials: include, sends HttpOnly session cookies)"]
    ServerCall --> ServerResp{"Server Session Valid & Role Matches?"}
    
    ServerResp -- "No / 403 Forbidden" --> RedirectForbidden["Redirect to /forbidden"]
    ServerResp -- "Yes / 200 OK" --> Tier3{"Tier 3: Profile Guard<br/>(ProfileSetupGuard.tsx)"}
    
    Tier3 --> CheckProfile["GET /security/checkAuth/profile-status"]
    CheckProfile --> ProfileDone{"profileComplete === true?"}
    
    ProfileDone -- "False" --> RenderSetupModal["Render Profile Setup Modal<br/>(Block Access Until Skills & Bio Completed)"]
    ProfileDone -- "True" --> RenderRoute["Render Protected Component"]
```

### Admin Multi-Factor Authentication (MFA)
In the administration portal ([Freelancer-Admin/src/pages/Login.tsx](file:///e:/DEVOPS/Freelancer-Admin/src/pages/Login.tsx)), a 3-step security verification is enforced:
1. **Credentials Verification**: Email and password encrypted with `xorEncrypt` submitted to `/api/vi/Manager/login`.
2. **Mandatory Password Rotation**: If `data.requiresPasswordChange === true`, access is blocked until a new secure password is set.
3. **Time-Based One-Time Password (TOTP)**: If 2FA is active, a 6-digit authenticator code or one-time emergency backup code is verified against `/admin/settings/2fa/verify`.

---

## 6. End-to-End Chat & Encryption Architecture

The real-time communication channel uses a hybrid transport and cryptography pipeline:

```mermaid
sequenceDiagram
    autonumber
    participant A as User A Browser (Sender)
    participant WS as WebSocket Gateway (:5000)
    participant B as User B Browser (Receiver)

    Note over A: 1. Anti-Circumvention Content Inspection
    A->>A: Regex Check: Phone Numbers (/\b\d{10,15}\b/) & Emails
    alt Contains Sensitive Contact Data
        A-->>A: Abort send; Trigger Alert: "Sharing contact information violates platform policy"
    else Message Clean
        Note over A: 2. Web Crypto API AES-GCM Encryption
        A->>A: Generate random 12-byte IV (ivHex)
        A->>A: crypto.subtle.encrypt(AES-GCM, key, messageText)
        A->>A: Format payload: "ivHex:encryptedTextHex:authTagHex"
        A->>WS: Send JSON: {sender, receiver, message: ciphertext, timestamp}
        WS->>WS: Broadcast to connected socket of receiver
        WS->>B: Deliver WebSocket packet
        Note over B: 3. Web Crypto API AES-GCM Decryption
        B->>B: Split payload by ":" -> [ivHex, cipherHex, authTagHex]
        B->>B: crypto.subtle.decrypt(AES-GCM, key, combinedBuffer)
        B->>B: Render Plaintext Message in UI
    end
```

---

## 7. Escrow, Milestone & Dispute State Machine

Financial flows are strictly modeled as state machines with irreversible transitions:

```mermaid
stateDiagram-v2
    [*] --> Draft: Acceptance of Project Bid
    Draft --> PendingSignature: Agreement Generated with SHA Content Hash
    PendingSignature --> Active: Both Client & Freelancer Sign
    
    Active --> EscrowFunded: Client Pays via Razorpay (create-order)
    
    state EscrowFunded {
        [*] --> MilestoneInProgress
        MilestoneInProgress --> MilestoneSubmitted: Freelancer Uploads Deliverable
        MilestoneSubmitted --> MilestoneRevision: Client Requests Revision (Max 3)
        MilestoneRevision --> MilestoneSubmitted
        MilestoneSubmitted --> MilestoneConfirmed: Client Approves Work
        MilestoneSubmitted --> AutoReleased: SLA Timeout (Auto-Release Timer)
        MilestoneConfirmed --> Released: Escrow Funds Disbursed (+ Early Bonus / - Late Penalty)
        AutoReleased --> Released
    }

    EscrowFunded --> DisputePendingPayment: Client or Freelancer Files Dispute
    DisputePendingPayment --> DisputeOpen: Initiator Pays Arbitration Fee via Razorpay
    
    state DisputeOpen {
        [*] --> EvidenceCollection: Parties Submit Files & Chat Logs
        EvidenceCollection --> AdminReview: Platform Moderator Inspects Case
        AdminReview --> ResolvedFullRefund: 100% Refunded to Client
        AdminReview --> ResolvedFullPayout: 100% Released to Freelancer
        AdminReview --> ResolvedSplit: Split Settlement (e.g., 60/40)
    }

    Released --> [*]: Project Completed
    ResolvedFullRefund --> [*]: Escrow Refunded
    ResolvedFullPayout --> [*]: Escrow Released
    ResolvedSplit --> [*]: Escrow Split
```

---

## 8. Summary of Architectural Strengths & Vulnerabilities

### Architectural Strengths
- **Clean Service Decoupling**: Frontend, admin, load balancer, and ML inference can be deployed, scaled, and updated independently.
- **Explainable AI Integration**: Matchmaking scores expose sub-feature weights, avoiding "black box" machine learning skepticism.
- **Multi-Layered Security**: End-to-end encrypted chat, dual-guard route access, XOR wire obfuscation, and SHA contract integrity checks.

### Architectural Risks (Addressed in Deep Dive)
- **Single Point of Failure at Edge**: The custom load balancer relies on in-memory state; restarting wipes rate-limiting tracking and active connection counts.
- **Connection Decrement Defect**: The load balancer fails to decrement server connections upon successful request completion.
- **Symmetric Key Exposure**: The chat encryption key is compiled into client-side Vite bundles (`VITE_ENCRYPTION_KEY`).

*(Full trade-off evaluations and technical mitigations are detailed in [06_TRADEOFFS_AND_DESIGN_DECISIONS.md](file:///e:/DEVOPS/PROJECT_DOCUMENTATION/06_TRADEOFFS_AND_DESIGN_DECISIONS.md) and [09_SECURITY_PERFORMANCE_AND_SCALABILITY.md](file:///e:/DEVOPS/PROJECT_DOCUMENTATION/09_SECURITY_PERFORMANCE_AND_SCALABILITY.md).)*
